# Deploy do Chatme no Coolify

Esta stack deve ser criada como um único recurso do tipo **Docker Compose
Empty**. O Coolify administra a rede interna, o domínio, o proxy HTTPS e os
volumes.

> **Pré-requisito:** uma instância do fzap já rodando (self-host própria ou
> contratada). O Chatme não substitui o conector de WhatsApp, ele depende dele.
> Veja [projeto-fzap](https://github.com/flouds-dncarbonell/projeto-fzap).

## Instalação

1. No projeto desejado, escolha **New Resource**.
2. Selecione **Docker Compose Empty**.
3. Cole o conteúdo de `docker-compose.yml` no editor.
4. Salve o Compose.
5. Em **Environment Variables**, informe:
   - `FZAP_BASE_URL`: URL pública da sua instância fzap;
   - `FZAP_ADMIN_TOKEN`: ADMIN_TOKEN dessa instância fzap;
   - `CHATME_LICENSE_KEY`: opcional — deixe vazio para rodar em modo
     não licenciado (funcional, com aviso, 1 tenant/1 pessoa).
6. Em **Domains**, confirme o domínio do serviço `chatme` e a porta interna
   `8081`.
7. Clique em **Deploy**.

O Coolify gera automaticamente:

- `SERVICE_URL_CHATME_8081`, que informa ao proxy a porta interna do Chatme;
- `SERVICE_URL_CHATME`, usada como `PUBLIC_BASE_URL` sem expor a porta interna;
- `SERVICE_PASSWORD_64_CHATMEMASTERKEY`, usada como `CHATME_MASTER_KEY`;
- `SERVICE_PASSWORD_64_STORAGESIGN`, usada como `STORAGE_SIGN_SECRET`;
- `SERVICE_PASSWORD_POSTGRES`, compartilhada pelo Chatme e PostgreSQL.

`CHATME_MASTER_KEY` e `STORAGE_SIGN_SECRET` não podem mudar depois do primeiro
boot — não regenere essas variáveis manualmente depois de colocar a instância
no ar.

## Primeiro acesso

Sem usuário administrador ainda, o Chatme mostra a tela de configuração
inicial: ela cria o primeiro superadmin e o primeiro tenant, sem precisar
mexer no banco.

## Persistência

Não remova os volumes ao recriar ou atualizar os containers:

- `chatme_pgdata`;
- `chatme_media`.

O PostgreSQL não publica porta no host. O Chatme acessa o banco pela rede
interna usando `DATABASE_URL` com host `postgres`.
