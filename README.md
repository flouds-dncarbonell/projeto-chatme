# Chatme - Inbox multi-tenant para WhatsApp, self-hosted e whitelabel

<div align="center">

![Chatme Version](https://img.shields.io/badge/Version-alpha-orange)
[![Docker](https://img.shields.io/badge/Docker-Available-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/dncarbonell/chatme)

**Inbox de atendimento multi-tenant sobre WhatsApp, com licenciamento e marca propria por cliente**

[Recursos](#-recursos) • [Deploy](#-deploy) • [Licenciamento](#-licenciamento) • [Aquisição](#-aquisição-do-chatme) • [Links](#-links-uteis)

</div>

---

## Sobre

O **Chatme** e uma inbox de atendimento multi-tenant para WhatsApp: cada empresa
(tenant) tem seus proprios canais, equipe e dados, isolados por linha (RLS) no
mesmo banco.

Conecta o WhatsApp via **fzap** (nao fala com o WhatsApp diretamente) e adiciona
os recursos de uma operacao de atendimento:

> **Pre-requisito:** o Chatme **precisa de uma instancia da fzap** rodando (self-host
> propria ou contratada) para conectar canais de WhatsApp — ela e o conector, o
> Chatme nao substitui nem embute esse papel. Veja
> [projeto-fzap](https://github.com/flouds-dncarbonell/projeto-fzap) (deploy) e
> [flouds.com.br/produtos/fzap](https://flouds.com.br/produtos/fzap) (produto).

- Multi-tenant com isolamento por linha (RLS) no Postgres
- Distribuicao de conversas para a equipe (fila, disponibilidade, escalonamento)
- Motor de regras e macros para automacao de atendimento
- Funil comercial (oportunidades) integrado a inbox
- Whitelabel por tenant: marca, cor, logo e e-mail transacional propios
- Self-host licenciado: um binario Go + Postgres/pgvector, sem dependencia de SaaS

---

## Recursos

- **Inbox completa**: conversas, midia (audio, video, documento, imagem), grupos,
  participantes, ficha de contato e organizacoes (empresas agrupando contatos).
- **Distribuicao de conversas**: disponibilidade do agente, expediente por canal,
  estrategias de atribuicao configuraveis pelo admin.
- **Motor de regras**: macros e acoes automaticas sobre conversas, catalogo de
  acoes exposto por API, log de execucao.
- **Funil comercial**: quadro de oportunidades ao lado da propria conversa.
- **Composer rico**: mencoes, formatacao e atalhos cientes do canal (WhatsApp).
- **Whitelabel por tenant**: nome do produto, logo, cor e e-mail com dominio
  proprio; a partir de duas cores o Chatme gera a rampa de cor completa.
- **Armazenamento de midia**: volume local (`fs`) ou S3-compativel (MinIO,
  Cloudflare R2, DO Spaces, AWS S3), configuravel por instalacao.
- **Transcricao de audio (opcional)**: OpenAI ou Groq, ativavel por tenant.
- **Onboarding sem seed manual**: primeira execucao abre tela de setup que cria
  o 1o superadmin e tenant.
- **Licenciamento e operacao multi-cliente**: um licenciado sobe uma instancia e
  hospeda varios clientes finais (tenants), suspendendo/reativando cada um sem
  afetar os outros.

---

## Deploy

### Imagem Docker

```bash
docker pull dncarbonell/chatme:alpha
```

### Execucao rapida (minima)

Precisa de um Postgres com a extensao `pgvector` (as migrations criam a extensao
e o schema sozinhas no boot) **e de uma instancia da fzap** ja no ar — e ela que
o Chatme usa para conectar canais de WhatsApp (`FZAP_BASE_URL`/`FZAP_ADMIN_TOKEN`
abaixo). Deploy da fzap: [projeto-fzap](https://github.com/flouds-dncarbonell/projeto-fzap).

```bash
docker run -d --name chatme-db \
  -e POSTGRES_USER=chatme -e POSTGRES_PASSWORD=troque-esta-senha -e POSTGRES_DB=chatme \
  pgvector/pgvector:pg16

docker run -d -p 8081:8081 \
  -e DATABASE_URL=postgres://chatme:troque-esta-senha@chatme-db:5432/chatme?sslmode=disable \
  -e PUBLIC_BASE_URL=https://app.suaempresa.com.br \
  -e FZAP_BASE_URL=https://fzap.suaempresa.com.br \
  -e FZAP_ADMIN_TOKEN=troque-este-token \
  -e CHATME_MASTER_KEY=$(openssl rand -hex 32) \
  -e STORAGE_SIGN_SECRET=$(openssl rand -hex 32) \
  -e SECURE_COOKIES=true \
  --link chatme-db \
  dncarbonell/chatme:alpha
```

Abra `PUBLIC_BASE_URL` no navegador: sem superadmin no banco, o Chatme mostra a
**tela de setup** (cria o 1o superadmin + tenant e ja loga).

### Stack completa

A stack completa (Swarm/Traefik/PostgreSQL/volumes/rede) esta neste arquivo:

- [stack.yml](https://github.com/flouds-dncarbonell/projeto-chatme/blob/main/stack.yml)

---

## Interfaces

- Console (inbox + configuracoes): `https://SUA-INSTANCIA/`
- API/admin (rotas JSON, sem UI propria): `https://SUA-INSTANCIA/admin/*`

---

## Licenciamento

O Chatme e **self-host licenciado**: sem `CHATME_LICENSE_KEY`, a instancia roda
em modo **unlicensed** (funcional, com aviso, limitado a 1 tenant e 1 pessoa,
sem canais novos e sem acoes de saida).

Com licenca ativa (Flouds entrega a `CHATME_LICENSE_KEY`), o licenciado pode
operar varios tenants (clientes finais), cada um com seus proprios canais,
equipe e marca, e suspender/reativar clientes individualmente.

---

## Aquisição do Chatme

Para **adquirir o Chatme** (licenca, implantacao e suporte):

- **Email:** `daniel@flouds.com.br`
- **WhatsApp:** `+55 51 99864-1731`

Se preferir, abra contato por este repositorio e enviamos proposta com o escopo
do seu ambiente.

---

## Links uteis

- Site do produto: [flouds.com.br/produtos/chatme](https://flouds.com.br/produtos/chatme)
- Docker Hub: [dncarbonell/chatme](https://hub.docker.com/r/dncarbonell/chatme)
- fzap (conector WhatsApp obrigatorio do Chatme): [projeto-fzap](https://github.com/flouds-dncarbonell/projeto-fzap) · [flouds.com.br/produtos/fzap](https://flouds.com.br/produtos/fzap)
