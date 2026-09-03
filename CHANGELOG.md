# Registro de ajustes

Histórico das decisões tomadas na montagem deste stack, com o motivo de cada
mudança. Serve para quem pegar o projeto depois entender por que ele está do
jeito que está — principalmente as partes que fogem do óbvio.

---

## 12 — Sessao que nao persiste: offline_access e SSO_AUTH_ONLY_NOT_SESSION

Depois de resolver o bloco do NPMplus, apareceu um conjunto de sintomas que
parecia bug de exibicao e nao era:

- login e criacao de item funcionam
- **listar** itens fica carregando pra sempre, na extensao **e** no celular
- o web vault lista normalmente, mas **F5 pede login de novo, sempre**
- logs do Vaultwarden: `200 OK` em tudo

Duas hipoteses foram descartadas antes da certa. Versao desatualizada da
extensao: era 2026.8.0, exatamente a exigida, e o mesmo comportamento aparecia
em Edge e Chrome. Cloudflare desafiando as requisicoes da extensao (os IPs de
cliente nos logs eram todos `172.64.0.0/13`, faixa da Cloudflare): contornar via
`hosts` apontando direto para o IP da VPS nao mudou nada.

A pista decisiva foi o F5. O problema nao era exibicao, era **sessao nao
renovando**. Criar funciona porque acontece logo apos o login, com token ainda
valido; listar depende de sincronizacao, que precisa renovar o token.

`SSO_AUTH_ONLY_NOT_SESSION=false` — escolhido deliberadamente para que revogar
no Entra derrubasse o acesso aqui — amarra a sessao ao ciclo de vida dos tokens
do IdP, e portanto **exige refresh token**. Refresh token exige `offline_access`
concedido em *API permissions* do App Registration do SSO, com admin consent.
Isso nunca foi confirmado durante o setup: a tela de permissoes revisada na
epoca era a do app de e-mail, nao a do SSO.

Mudancas:

- `SSO_AUTH_ONLY_NOT_SESSION` virou `VW_SSO_AUTH_ONLY_NOT_SESSION` (default
  `false`), para permitir desacoplar a sessao do IdP sem editar o compose. O
  custo de `true` esta documentado: perde-se a revogacao imediata pelo Entra.
- README ganhou a secao "Sessao que nao persiste" com o conjunto completo de
  sintomas, porque a combinacao e dificil de interpretar isoladamente — cada
  sintoma sozinho aponta para o lugar errado.
- Registrado tambem que preencher apenas a **URL do servidor** no app de celular
  e o correto (ele deriva API, identidade, cofre web e icones), para nao virar
  falso suspeito numa proxima investigacao.

---

## 11 — Causa raiz do loop: bloco Advanced do NPMplus ausente

O `extra_hosts` do item 10 não resolveu o loop. Os logs do `key-connector`
mostravam a busca do JWKS funcionando (`discovered identity provider ...
jwks_uri=...`) — então hairpin NAT não era a causa desta vez, embora a correção
continue válida para quando for.

Diagnóstico decisivo veio de comparar as duas pontas:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8281/alive        # 200
curl -sS -o /dev/null -w '%{http_code}\n' https://<dominio>/keyconnector/alive  # 404
```

KC saudável localmente, rota pública ausente: o bloco `location /keyconnector/`
do passo 8 nunca tinha sido colado no NPMplus. Ele mora na aba **Advanced**,
separada dos campos principais do proxy host (domínio, scheme, porta) — dá para
configurar o proxy host inteiro sem notar que essa aba tinha algo pendente.

A primeira tentativa de diagnóstico rodou o `curl` de teste no `cmd.exe` do
Windows em vez da VPS, o que mascarou o sinal: `/dev/null` não existe ali e as
aspas simples do `-w` não são removidas como num shell POSIX, produzindo um erro
de escrita em vez de uma resposta limpa. O código HTTP real (404) apareceu
mesmo assim, entre aspas literais — dava para ler, mas exigiu explicar por que o
teste era pouco confiável antes de confiar no número.

README ganhou uma seção "Confirme antes de testar o login" logo após o bloco
Advanced, com os dois curls lado a lado e a advertência de rodá-los na VPS, não
no Windows. O objetivo é pegar esse gap em segundos da próxima vez, não depois
de uma sessão inteira de depuração comparando logs.

---

## 10 — extra_hosts por padrão no Key Connector (hairpin NAT)

Um login via SSO passou a "funcionar e voltar pro início" de forma silenciosa:
os logs do Vaultwarden mostravam `200 OK` em toda a sequência — login, sync,
`get_confirmation_details` — sem nenhum erro. O navegador simplesmente parava de
progredir depois do último `200` e, minutos depois, reiniciava o fluxo do zero.

Diagnóstico: nada daquilo passava pelo Vaultwarden. O próximo passo do fluxo é o
**navegador** chamar o Key Connector diretamente
(`KEY_CONNECTOR_URL + "/user-keys"`), e essa chamada não aparece no log do
Vaultwarden. Causa raiz: `KC_IDENTITY_AUTHORITY` aponta para o próprio domínio
público — o Key Connector busca o JWKS ali para validar tokens, o que exige o
container sair para o IP público da VPS e voltar por hairpin NAT. Provedor sem
suporte a isso = busca do JWKS falha = toda validação de token falha = a
inscrição no Key Connector nunca completa, sem gerar log de erro em lugar nenhum
que o operador olharia primeiro.

Esse cenário já estava previsto no troubleshooting desde o item 3
(caminho único), mas como algo que o operador aplicaria manualmente se
precisasse. Passou a vir **por padrão** no serviço `key-connector`:

```yaml
extra_hosts:
  - "${VW_DOMAIN_HOST}:host-gateway"
```

Exigiu uma variável nova, `VW_DOMAIN_HOST` — só o hostname, sem `https://` e sem
barra. O compose não consegue derivar isso de `VW_DOMAIN` sozinho (não há
manipulação de string em interpolação de variável do Compose), então os dois
precisam ser mantidos em sincronia manualmente. Marcada `[OBRIGATÓRIA]` no
`.env.example`.

Se o provedor já suporta hairpin NAT nativamente, esta entrada é inofensiva —
só evita uma viagem de rede desnecessária a cada busca de JWKS.

---

## 9 — Build sem atestação e fixação por digest

Um redeploy falhou puxando a imagem do Key Connector:

```
failed to do request: Get ".../manifests/sha256-3fd08543...": dial tcp 140.82.114.34:443: i/o timeout
```

O digest procurado não era o da imagem — era o **manifest de atestação** que o
buildx anexa por padrão, publicado como uma entrada de plataforma
`unknown/unknown` dentro de um índice OCI. Diferente dos erros anteriores, este
não era permissão: `busybox` e a imagem do Vaultwarden, essa também no ghcr,
baixaram no mesmo run.

A imagem passou a ser construída com `--provenance=false --sbom=false`. O
resultado é um manifest único, sem índice e sem a entrada extra. O digest do
conteúdo não mudou (`sha256:e1dd0306...` antes e depois) — só sumiu o invólucro,
o que confirma que o binário é o mesmo.

Proveniência não é validada em nenhum ponto deste fluxo, então era uma
requisição por pull que só tinha como atrapalhar. Também explica o
`version_count: 3` que o pacote mostrava para uma tag só.

**`KC_IMAGE` passou a ser fixado por digest**, como o `VW_IMAGE` já era. O
episódio deixou claro que tag é mutável inclusive no registry da própria
empresa: republicar `e784bd8` trocou o digest para o qual ela apontava, com o
mesmo commit de origem.

---

## 8 — Correção do healthcheck do Vaultwarden

Primeiro deploy falhou com `dependency failed to start: container vaultwarden is
unhealthy`. O servidor estava de pé; a sonda é que estava errada.

O healthcheck usava `wget -qO- http://127.0.0.1:8080/alive`, herdado de quando o
stack apontava para `vaultwarden/server:1.37.2-alpine`. A **imagem do fork é
Debian 13**, não Alpine, e Debian não traz `wget` — só `curl`. O comando falhava
por binário inexistente, o container era marcado unhealthy, e o `key-connector`
não subia por depender de `service_healthy`.

Passou a usar o `/healthcheck.sh` que a própria imagem traz, que é mais correto
que a sonda manual: lê `ROCKET_PORT`, troca `0.0.0.0` por `localhost`, deriva o
base path a partir do `DOMAIN` e usa `curl`.

Segunda vez que uma suposição sobre a imagem vira bug — a primeira foi o GID
(item 6). Ambas teriam sido evitadas inspecionando a imagem em vez de deduzir do
Dockerfile ou da tag de origem:

```bash
docker run --rm --entrypoint sh <imagem> -c 'cat /etc/os-release; command -v curl wget'
```

---

## 7 — Sanitização e pacote público no registry

**Documentação sanitizada.** Todos os domínios, e-mails, IPs, nomes de
organização e caminhos passaram a usar valores fictícios — `example.com`,
`Example Corp`, `100.64.0.10`, `/opt/vaultwarden`. O README ganhou uma tabela
"Valores de exemplo" mapeando cada placeholder ao que precisa ser substituído.

Nenhum identificador real chegou a ser versionado: tenant ID, client ID e IPs de
produção só existiram no `.env`, ignorado desde o primeiro commit.

**Pacote do Key Connector passa a ser público** no ghcr, e a VPS deixa de
precisar de credencial de registry.

A razão é uma limitação concreta do GitHub: **o GHCR não aceita fine-grained
personal access tokens.** A documentação oficial afirma que "GitHub Packages only
supports authentication using a personal access token (classic)", e o suporte a
fine-grained foi retirado do roadmap público em 2024, sem substituto.

Isso significa que manter o pacote privado exigiria um token clássico de **conta
inteira** — sem escopo por pacote — armazenado em `~/.docker/config.json` em
base64 não criptografado, com validade a controlar e preso à conta pessoal de
quem montou o stack. A imagem, por outro lado, é build de código AGPL público e
não contém segredo algum.

Trocar "esconder qual software rodamos" por essa credencial não compensa. O
caminho privado segue documentado no README para quem precise, junto da
recomendação de migrar para organização com conta de serviço.

---

## 6 — Correção do GID do Key Connector

O compose usava `user: "10001:10001"` e `chown 10001:10001`. Errado: a imagem
real roda como `uid=10001(keyconnector) gid=999(keyconnector)`.

A dedução original veio do `useradd -r -u 10001 keyconnector` do Dockerfile —
mas `-u` fixa só o UID; o GID sai do próximo livre do sistema. Só apareceu ao
inspecionar a imagem construída:

```bash
docker run --rm --entrypoint id ghcr.io/<seu-usuario>/key-connector:e784bd8
```

Duas mudanças:

- `user: "10001:999"`, conferindo com a imagem.
- `init-perms` passa a fazer `chown` **apenas do dono** (`chown 10001`), com
  modo `700` em vez de `750`. Assim o acesso continua correto mesmo que um
  rebuild futuro atribua outro GID — o processo acessa como dono, e o grupo
  deixa de importar.

Vale como lembrete: assumir UID/GID a partir da leitura de um `Dockerfile` não
substitui inspecionar a imagem construída.

---

## 5 — SMTP deixa de bloquear o deploy; bootstrap sem e-mail

Descoberta durante o preenchimento: **o Vaultwarden não implementa OAuth para
SMTP.** A `lettre` suporta XOAUTH2, mas o servidor não obtém nem renova o token,
que expira em uma hora. As variáveis `SMTP_XOAUTH2_*` existem só como proposta
em discussões abertas. Logo, não há como entregar app id + client secret direto
ao Microsoft 365.

Quem quer credencial de aplicativo em vez de senha de pessoa tem como saída o
**Azure Communication Services**: relay SMTP onde o usuário é
`<recurso-ACS>.<AppId>.<TenantId>` e a senha é o client secret. Do ponto de
vista do Vaultwarden é SMTP AUTH comum, que ele suporta. Documentado como opção
A na etapa F do `PREENCHENDO-ENV.md`, com a ressalva de usar um App Registration
separado do SSO.

Duas mudanças no compose por causa disso:

**SMTP virou opcional.** Todo o bloco usa `${VAR:-}` — `SMTP_HOST` vazio desliga
o envio e o servidor sobe normal. Provisionar ACS com verificação de domínio
leva dias, e não faz sentido o cofre inteiro esperar por e-mail.

O critério para o que trava o deploy ficou mais claro: trava o que falha em
**silêncio**. `ADMIN_TOKEN` vazio desabilita o painel sem avisar;
`ORG_CREATION_USERS` vazio afrouxa permissão sem avisar. SMTP quebrado aparece
no primeiro convite que não chega — não precisa do guarda-corpo.

**`SIGNUPS_ALLOWED` virou variável** (`VW_SIGNUPS_ALLOWED`, default `false`).
Sem isso havia um impasse: sem SMTP não chega convite, sem convite não se cria a
primeira conta, e sem conta não se configura nada. Ligando temporariamente, o
login via SSO cria a conta sozinho. A exposição na janela é limitada pela
whitelist de domínio e pelo `Assignment required` do Entra.

---

## 4 — Preparo para o deploy remoto pelo Dockhand

Duas mudanças, ambas para o modelo "transfere o compose e sobe o stack", em que
ninguém entra na VPS antes.

**Variáveis obrigatórias falham alto.** Todas as variáveis sem default agora
usam `${VAR:?mensagem}`. Se o Dockhand não entregar alguma, o deploy aborta
nomeando a variável, em vez de subir um container com configuração vazia. Isso
ataca diretamente o risco que o padrão da casa descreve — "no deploy remoto ela
pode chegar vazia sem erro" — e alguns desses vazios seriam graves em silêncio:

- `ADMIN_TOKEN` vazio **desabilita o painel** `/admin`.
- `ORG_CREATION_USERS` vazio deixa **qualquer usuário criar organização**.
- `KC_ENCRYPTION_KEY` vazia compromete o serviço que guarda as chaves.

Variáveis com default sensato usam `${VAR:-valor}` e continuam opcionais
(`SMTP_PORT`, `SMTP_SECURITY`, `DATABASE_MAX_CONNS`, `PUSH_*`, entre outras). O
`.env.example` marca cada uma como `[OBRIGATÓRIA]` ou `[opcional]`.

Para conferir sem subir nada: `docker compose config --quiet`.

**Serviço `init-perms`.** Um container busybox que roda como root antes dos
demais, ajusta o dono dos diretórios de dados (1000 e 10001) e sai.

Existe porque os bind mounts em `/opt/vaultwarden/*` são criados pelo
Docker como `root` quando não existem, e os containers rodam como usuário
não-root — resultado: crash-loop por permissão no primeiro deploy. Antes disso
era um passo manual de SSH, incompatível com deploy remoto.

Ele ajusta só o diretório raiz, não recursivamente, para não reescrever a posse
de todos os anexos a cada deploy. A contrapartida é que conteúdo já existente
com dono errado precisa de um `chown -R` manual — documentado no README.

---

## 3 — Key Connector em caminho único, sem subdomínio

**Antes:** `keyconnector.example.com`, proxy host separado no NPMplus.
**Agora:** `vault.example.com/keyconnector`, um `location` no mesmo proxy host.

Motivo: um DNS a menos, um proxy host a menos, e o web vault passa a chamar o
Key Connector em same-origin — elimina CORS e a entrada extra no CSP.

A mudança só é segura porque foi verificada nos dois lados antes de aplicar:

- **Cliente** (`bitwarden/clients`, `libs/common/src/services/api.service.ts`):
  a URL é montada por concatenação pura, sem normalização —
  `createRequest(keyConnectorUrl + "/user-keys", ...)`. Um caminho na URL base
  sobrevive à concatenação.
- **Servidor** (PR #7419, `src/config.rs`): a validação exige apenas URL com
  protocolo, sem impor origin puro. O CSP usa só `u.origin()`, que aqui é o
  próprio domínio do Vaultwarden.
- **Servidor:** não faz nenhuma requisição para essa URL — só a entrega aos
  clientes em `KeyConnectorUrl`. Quem chama é sempre o cliente.

Regras que nasceram daí:

- `VW_KEY_CONNECTOR_URL` **nunca** termina com barra, ou a concatenação gera
  `//user-keys`.
- No NPMplus, `location /keyconnector/` **e** `proxy_pass .../` precisam das
  duas barras finais. Sem a do `proxy_pass`, o prefixo não é removido e o
  serviço responde 404.

Também ficou registrado no README que o Key Connector atende **todos** os
clientes (web vault, extensão, desktop, mobile, CLI), não só a extensão — o
método vive em `libs/common`, compartilhada por todos. Se ele cair, ninguém abre
o cofre.

---

## 2 — Key Connector: login sem senha mestra

**Antes:** Vaultwarden upstream `1.37.2-alpine`, SSO com Entra ID, senha mestra
obrigatória.
**Agora:** fork com o [PR #7419](https://github.com/dani-garcia/vaultwarden/pull/7419)
+ serviço [acul021/key-connector](https://github.com/acul021/key-connector).

A versão anterior deste stack afirmava que masterless SSO só existia no Bitwarden
Enterprise e que não havia equivalente para Vaultwarden. **Isso estava errado.**
O `acul021/key-connector` é esse equivalente: uma reimplementação independente em
Rust (Axum + SQLx), com o protocolo deduzido do código aberto do Vaultwarden e
dos clientes Bitwarden.

O que entrou junto com a decisão:

- **Imagem de fork** `ghcr.io/acul021/vaultwarden`, porque o PR #7419 está
  aberto, não mergeado. **Somente amd64.** Pinada por digest — `:testing` é tag
  móvel e não serve para produção.
- **Imagem do Key Connector construída internamente**, a partir de commit fixo,
  publicada no registry da empresa. O repositório só publica `Dockerfile`. Para um
  serviço que guarda as chaves de todo mundo, controlar a cadeia de suprimento
  vale mais que a conveniência de puxar imagem de terceiro.
- **Volume próprio para o KC.** O projeto compila o `sqlx` só com o backend
  SQLite — não há como apontá-lo para o PostgreSQL da VPS de banco.
- **UID 10001** no container do KC, conforme o `useradd -r -u 10001` do
  Dockerfile do projeto.
- **`SSO_ONLY=true`** como padrão: usuários inscritos não têm senha mestra, então
  o login local precisa estar fechado.

### O que mudou no modelo de segurança

Com senha mestra, a chave nunca existe no servidor. Com Key Connector, ela fica
guardada no serviço, cifrada em AES-256-GCM sob a `KC_ENCRYPTION_KEY`. Quem
obtiver o banco do KC **e** a chave **e** um token OIDC válido decifra os cofres.

Isso é inerente ao desenho do Key Connector — vale igual no Bitwarden oficial.
O que é diferente aqui é a maturidade: versão 0.1.0, um mantenedor, sem auditoria
externa.

Daí as duas regras operacionais registradas no README:

1. `KC_ENCRYPTION_KEY` e o banco do KC **não podem ficar no mesmo backup**.
   Juntos, o backup vira cópia em claro de todos os cofres.
2. Perder qualquer um dos dois = **todos os cofres permanentemente
   indecifráveis**. Não é "usuários fazem login de novo".

Owner e Admin não podem se inscrever no Key Connector (restrição do próprio PR),
então seguem com senha mestra — o que dá um caminho de emergência natural.

O README documenta o rollback para o upstream estável, que é barato **só
enquanto poucas contas estiverem inscritas**.

---

## 1 — Remoção do backup para Cloudflare R2

**Antes:** serviço `vaultwarden-backup` com `rclone` sincronizando `/data` para o R2.
**Agora:** removido.

O Vaultwarden **gera anexos sim**, em `/data/attachments`, cifrados no cliente e
referenciados no PostgreSQL. São recuperáveis desde que `/data` e o banco sejam
restaurados juntos — e `/data` já entra na rotina de backup da VPS. O R2 seria
conveniência de destino, não requisito.

O backup passou a ser documentado como três itens segregados: banco PostgreSQL,
`/data`, e o banco do Key Connector — com a `KC_ENCRYPTION_KEY` guardada
separadamente.

---

## 0 — Como o padrão de infraestrutura foi aplicado

O documento interno de padrões de infraestrutura da empresa (não versionado aqui)
foi escrito para orientar devs criando sistemas
novos. Aqui o software é pronto e de terceiro, então o que valeu foi o que
descreve a **infraestrutura**, não as regras de codificação.

Aproveitado:

- Role e database dedicados no PostgreSQL, via IP `100.64.x.x` da tailnet, com
  `DATABASE_MAX_CONNS=10` limitando o pool.
- Tudo publicado em `127.0.0.1`; NPMplus segue como único componente com portas
  públicas.
- Configuração só por variável de ambiente, listada explicitamente no
  `environment:`, sem `.env` versionado e sem segredo no repositório.
- Container não-root, `read_only: true`, `cap_drop: ALL`, `no-new-privileges`,
  imagem com tag fixa (aqui, digest).

Não aplicável: R2 (o Vaultwarden não fala S3), `/api/health` (o endpoint é
`/alive`), logs em JSON (o Vaultwarden só emite texto) e Dockerfile multi-stage
próprio.

Observação operacional que continua valendo: o Vaultwarden roda as migrations no
boot e isso não é configurável. **Nunca escalar além de 1 réplica** — duas
instâncias subindo juntas corrompem o schema.

### Detalhes do Entra ID que custam horas se passarem batido

- **`SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION=true` é obrigatório.** O Entra não
  emite a claim `email_verified` e sem isso *todo* login é recusado. Vem
  obrigatoriamente com `SSO_SIGNUPS_MATCH_EMAIL=false`, senão abre brecha de
  associação de conta.
- **Optional claim `email` no ID token.** Se o UPN divergir do e-mail, o login
  entra em loop silencioso, sem mensagem de erro.
- **Websockets ligados no NPMplus**, senão o `/notifications/hub` não conecta e
  nada sincroniza entre dispositivos.
