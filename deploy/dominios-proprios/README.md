# Domínios próprios das contas

A partir da versão **0.7.0**, cada conta (tenant) do Chatme pode atender no
domínio do próprio cliente, como `atendimento.cliente.com.br`. O servidor é
preparado uma vez; depois, cada domínio novo é cadastrado no console e o Chatme
verifica o DNS, publica a rota no Traefik e pede o certificado HTTPS sozinho.
Nenhuma stack precisa ser editada por cliente.

`PUBLIC_BASE_URL` e a licença continuam no domínio principal da instalação.
Domínios das contas não mudam a licença nem os limites do plano.

## Requisitos

- Chatme `0.7.0` ou mais recente
- Traefik 2.11 ou 3, com resolver ACME funcionando e certificados em volume
- Acesso à configuração estática do Traefik (argumentos `command` ou `traefik.yml`)
- DNS **sem proxy de CDN** (na Cloudflare, nuvem cinza), no domínio principal e
  nos domínios dos clientes

## Preparação do servidor (uma vez)

Os nomes abaixo seguem o [`stack.yml`](../../stack.yml) deste repositório,
publicado como stack `chatme` na rede `proxy`.

1. **Domínio principal.** O host de `PUBLIC_BASE_URL` precisa de um registro A
   para o IPv4 público da VPS. Crie AAAA só se a VPS também atende por IPv6.
   Esse host é o destino do CNAME dos clientes.

2. **Segredo.** Gere um segredo para a comunicação entre Traefik e Chatme. Ele
   vive só nas definições das stacks, nunca em repositório:

   ```bash
   openssl rand -hex 32
   ```

3. **Stack do Chatme.** Descomente as variáveis de domínios próprios no
   `stack.yml` e preencha:

   ```yaml
   environment:
     - PUBLIC_BASE_URL=https://app.suaempresa.com.br
     - TRUST_PROXY_HEADERS=true
     - CUSTOM_DOMAINS_IPS=203.0.113.10
     - CUSTOM_DOMAINS_PROXY_TOKEN=SEU_SEGREDO
     - CUSTOM_DOMAINS_UPSTREAM_URL=http://chatme_chatme:8081
   ```

   - `CUSTOM_DOMAINS_IPS`: IPs públicos da VPS, separados por vírgula. Todo IP
     publicado no DNS de um cliente precisa estar nesta lista.
   - `CUSTOM_DOMAINS_UPSTREAM_URL`: endereço interno que o Traefik usa para
     chegar ao Chatme. No Swarm é `<stack>_<serviço>`; no Docker Compose,
     normalmente `http://chatme:8081`. Sem caminho, credenciais ou query string.
   - `TRUST_PROXY_HEADERS=true` é obrigatório: o Traefik termina o HTTPS e
     repassa HTTP ao Chatme. Mantenha o serviço sem porta publicada.
   - As três `CUSTOM_DOMAINS_*` vão juntas. Preencher só parte delas impede o
     boot, com a variável errada indicada no log. Sem nenhuma, o Chatme fica em
     modo manual.

4. **Stack do Traefik.** Acrescente o provider HTTP aos argumentos existentes,
   sem remover os providers, entradas, resolver e redes que já estão lá:

   ```yaml
   command:
     # Acrescentar aos argumentos existentes:
     - --providers.http.endpoint=http://domains:SEU_SEGREDO@chatme_chatme:8082/traefik
     - --providers.http.pollinterval=5s
     - --providers.http.polltimeout=5s
   ```

   Se o Traefik usa arquivo estático:

   ```yaml
   providers:
     http:
       endpoint: "http://domains:SEU_SEGREDO@chatme_chatme:8082/traefik"
       pollInterval: 5s
       pollTimeout: 5s
   ```

   O usuário é literalmente `domains` e a senha é o mesmo segredo do passo 2.
   [`traefik.yml`](traefik.yml) é a stack completa gerada pelo instalador da
   Flouds, já com essas linhas, para servir de referência.

   - Traefik e Chatme precisam estar na mesma rede interna (`proxy`).
   - **Não publique a porta 8082** e não crie rota pública para `/traefik`.
   - O Traefik aceita um único provider HTTP. Se outro serviço já usa um,
     combine as configurações antes; trocar o endpoint removeria as rotas dele.

5. **Nomes do Traefik.** O Chatme usa estes padrões. Se o seu Traefik usa
   outros nomes, informe na stack do Chatme:

   | Variável | Padrão |
   |---|---|
   | `CUSTOM_DOMAINS_HTTP_ENTRYPOINT` | `web` |
   | `CUSTOM_DOMAINS_ENTRYPOINT` | `websecure` |
   | `CUSTOM_DOMAINS_CERT_RESOLVER` | `letsencryptresolver` |
   | `CUSTOM_DOMAINS_LISTEN` | `:8082` |

   As portas 80 e 443 precisam chegar ao Traefik, inclusive para o desafio ACME.

6. **Atualize as duas stacks.** Em **Console → Contas → conta → Editar
   domínios**, o aviso de conexão pendente some quando o Traefik consultar o
   Chatme (a cada 5 segundos).

O console também mostra estes trechos em **Preparar a instalação → Ver
instruções da stack** e copia um texto pronto para a IA que opera o servidor em
**Copiar instruções para IA**.

## Para cada cliente

Cadastre o domínio em **Console → Contas → conta → Editar domínios**. O painel
mostra o registro que o dono do domínio cria no provedor de DNS dele:

| Tipo | Nome | Destino |
|---|---|---|
| CNAME | `atendimento.cliente.com.br` | `app.suaempresa.com.br` |

Alguns provedores pedem só `atendimento` no campo de nome. Para domínio raiz
(`cliente.com.br`), use o registro A/AAAA que o painel indica, normalmente com
`@`. Preserve os registros de e-mail. O Chatme não acessa o DNS do cliente.

| Estado | O que fazer |
|---|---|
| Aguardando DNS | O registro ainda não aponta para os IPs configurados. Conferir o DNS, inclusive IPv6, e aguardar a propagação. |
| Preparando HTTPS | DNS certo; o certificado está sendo emitido. Se demorar, conferir nomes das entradas, resolver, portas, CAA e logs do ACME. |
| Ativo | O endereço responde com HTTPS válido. |
| Conta suspensa | O endereço continua reservado, mas a rota sai do ar. |
| Configuração manual | A instalação ainda não foi preparada (variáveis ausentes). |
| Endereço reservado | É o domínio principal ou o destino do CNAME; escolha outro. |

Regras:

- Cada conta aceita **um** domínio. Trocar o endereço desliga o anterior, sem
  redirecionamento.
- No domínio próprio, o login entra direto naquela conta e o seletor de contas
  fica indisponível. Console e API de plataforma usam o domínio principal.
- Convites, recuperação de senha, compartilhamentos e e-mails passam a usar o
  domínio da conta quando ele está ativo.

## Coolify, Easypanel e modo manual

Esses painéis gerenciam o próprio proxy e podem não permitir editar a
configuração estática do Traefik. Nesse caso, deixe as três `CUSTOM_DOMAINS_*`
vazias: o Chatme fica em **configuração manual**. Cadastre o domínio na conta
pelo console e adicione o mesmo domínio ao serviço `chatme` no painel, que
cuida do DNS e do HTTPS.

## API de plataforma

O mesmo fluxo existe para automações, com `Authorization: Bearer ct_pt_...` de
um titular superadmin:

- `GET /platform/api/v1/tenants/{tenant}/domains`: escopo `tenants:read`
- `PUT /platform/api/v1/tenants/{tenant}/domains`: escopo `tenants:write`,
  corpo `{"domains":["atendimento.cliente.com.br"]}` (zero ou um item)
- `POST /platform/api/v1/tenants/{tenant}/domains/check`: escopo `tenants:write`

## Atualizando de versões anteriores

A migração mantém só o primeiro domínio de cada conta e registra a lista
anterior na auditoria. Revise contas com mais de um domínio antes de atualizar
e faça backup do banco. `CUSTOM_DOMAINS_TARGET` deixou de ser lida: o destino
do CNAME agora é sempre o host de `PUBLIC_BASE_URL`.
