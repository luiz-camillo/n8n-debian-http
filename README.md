# n8n-debian-http
Documentação e arquivos de configuração para rodar n8n + Docker + Postgres 17 + Nginx no Debian 13 (Trixie). Sem TLS/SSL.
Formato: Laboratório (LAB) | Segurança: HTTP Puro (Porta 80 - Sem TLS/SSL)

Documentação completa e arquivos de orquestração prontos para rodar o n8n (Self-Hosted) em ambiente de produção ou testes no Debian 13 (Trixie). Ideal para cenários onde o certificado SSL é gerenciado por uma camada externa (como Cloudflare, VPN ou Load Balancer).

A Stack do Lab
SO: Debian 13 (Trixie) - Sistema operacional base

Orquestração: Docker + Docker Compose - Isolamento e gerenciamento de containers

Automação: n8n (Self-hosted) - Motor de workflows e integrações

Banco de Dados: PostgreSQL 17 - Persistência estável de dados e logs

Proxy Reverso: Nginx - Roteamento do tráfego web para a porta interna

Firewall: nftables - Proteção nativa e restrição de acessos por IP

Conteúdo do Repositório
A documentação principal traz o passo a passo e os blocos de configuração estruturados de forma modular:

Preparação do SO: Instalação limpa e padronizada do Docker oficial no Debian 13.

Orquestração (docker-compose.yml): Integração do n8n com o PostgreSQL 17, isolando as senhas em um arquivo .env seguro.

Proxy Reverso (nginx.conf): Configuração em Nginx para receber o tráfego da porta 80 e repassar ao container.

Segurança com Firewall (nftables.conf): Regras prontas para trancar o servidor, liberando o painel estritamente para os IPs da sua equipe.

Mapeamento de Backups: Identificação exata de quais pastas e volumes você precisa salvar para nunca perder suas automações.

Como Usar
Acesse o arquivo de documentação principal contido neste repositório.

Ajuste as variáveis de ambiente, alterando as senhas do banco e definindo os IPs autorizados no firewall.

Siga o passo a passo, copiando e colando os comandos sequencialmente direto no terminal do seu servidor.
