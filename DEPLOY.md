# Instalação n8n Laboratório — Docker + PostgreSQL + Nginx (Debian 13)

# Deploy do n8n self-hosted via Docker, com PostgreSQL 17 como banco e Nginx como proxy reverso, em Debian 13 (Trixie). Setup em **HTTP puro** (sem TLS/SSL) — n8n e webhook expostos na porta 80.

## Stack

| Camada | Tecnologia |
|---|---|
| SO | Debian 13 (Trixie) |
| Orquestração | Docker + Docker Compose |
| Automação | n8n (self-hosted) |
| Banco de dados | PostgreSQL 17 |
| Proxy reverso | Nginx |
| Firewall | nftables |

## ⚙️ Instalação

### Pré-requisitos
- VM Debian 13 (Trixie) com acesso root
- IP público ou domínio apontado para a VM

### 1. Atualizar o sistema

```bash
apt update && apt upgrade -y
apt install -y ca-certificates curl gnupg git
```

### 2. Instalar o Docker

```bash
install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
docker --version
```

### 3. Preparar diretório e persistência de dados

```bash
mkdir -p ~/n8n-docker/n8n_data
cd ~/n8n-docker
chown -R 1000:1000 ./n8n_data   # UID 1000 = usuário node dentro do container n8n
```

### 4. Configurar variáveis de ambiente

```bash
nano .env
```

`.env`:
```env
POSTGRES_PASSWORD=sua_senha_forte_aqui
```

```bash
chmod 600 .env
```

> O `docker compose` lê o `.env` automaticamente quando ele está na mesma pasta do `docker-compose.yml` — a senha nunca fica exposta no arquivo de orquestração.

### 5. Orquestração dos containers

```bash
nano docker-compose.yml
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:17
    container_name: n8n_postgres
    restart: always
    environment:
      TZ: America/Sao_Paulo
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:2.28.4
    container_name: n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    depends_on:
      - postgres
    environment:
      - TZ=America/Sao_Paulo
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_PORT=5678
      - N8N_SECURE_COOKIE=false
      - NODE_ENV=production
      - WEBHOOK_URL=http://SEU_IP_OU_DOMINIO/
    volumes:
      - ./n8n_data:/home/node/.n8n

volumes:
  postgres_data:

networks:
  default:
    ipam:
      config:
        - subnet: 172.18.0.0/16
```

```bash
docker compose up -d
docker ps
```

### 6. Proxy reverso (Nginx)

```bash
apt install -y nginx
systemctl enable --now nginx
nano /etc/nginx/sites-available/n8n
```

`/etc/nginx/sites-available/n8n`:
```nginx
server {
    listen 80;
    server_name SEU_IP_OU_DOMINIO;

    client_max_body_size 20M;
    proxy_read_timeout 300;
    proxy_send_timeout 300;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
rm -f /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### 7. Firewall (nftables)

```bash
nano /etc/nftables.conf
```

`/etc/nftables.conf`:
```
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    set autorizados {   # IPs autorizados (equipe/consultoria)
        type ipv4_addr
        flags interval
        elements = { SEU_IP_1, SEU_IP_2, SEU_IP_3 }
    }

    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        iif lo accept
        ip protocol icmp counter accept
        ip saddr @autorizados counter accept
        tcp dport { 80 } accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        ip saddr 172.18.0.0/16 ct state new accept
    }

    chain output { type filter hook output priority filter; policy accept; }
}
```

```bash
nft -f /etc/nftables.conf
systemctl restart docker     # reinjeta o NAT após mexer no firewall
systemctl enable nftables
```

### 8. Teste de conectividade com a OpenAI

```bash
docker exec n8n node -e "fetch('https://api.openai.com').then(r => console.log('Sucesso! Status:', r.status)).catch(e => console.error('Falha na Conexão:', e.message))"
```

## 🔒 Segurança

- Senha do PostgreSQL isolada em `.env` (não versionado), referenciada via `${POSTGRES_PASSWORD}` no compose.
- Firewall restringe entrada apenas aos IPs da equipe/consultoria via `nftables`.
- Porta do n8n (`5678`) exposta apenas em `127.0.0.1` — acesso externo só via Nginx.
- **Nunca commitar** `.env`, chaves de API ou IPs reais de clientes/infraestrutura neste repositório.

## 💾 Backup

Dois caminhos críticos devem ser incluídos na rotina de backup do servidor:

| Caminho | Conteúdo |
|---|---|
| `~/n8n-docker/n8n_data` | Credenciais, chave de criptografia e definições de workflows do n8n |
| Volume `postgres_data` | Banco relacional completo (execuções, histórico) |

## 📄 Licença

Este projeto está licenciado sob a [MIT License](LICENSE).

---

**Autor:** Luiz Ferreira
