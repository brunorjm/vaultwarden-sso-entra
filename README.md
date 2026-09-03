# Vaultwarden + Key Connector

Cofre de senhas para os departamentos do escritório, com login pela conta
corporativa Microsoft (Entra ID, OAuth 2.0 / OIDC) **sem senha mestra**, usando
o [acul021/key-connector](https://github.com/acul021/key-connector).

| Componente | Imagem | Porta local | Endereço público |
|---|---|---|---|
| Vaultwarden | fork com PR #7419 (`VW_IMAGE`) | `127.0.0.1:8222` | `vault.example.com` |
| Key Connector | build própria (`KC_IMAGE`) | `127.0.0.1:8281` | `vault.example.com/keyconnector` |
| PostgreSQL | VPS de banco, via tailnet | — | — |

Um domínio só. O Key Connector é publicado como caminho no mesmo host, não
como subdomínio — o porquê está em "Por que um caminho, e não um subdomínio".

> **Primeira vez aqui?** [PREENCHENDO-ENV.md](PREENCHENDO-ENV.md) passa por cada
> variável do `.env` na ordem em que dá para conseguir os valores. Este README
> está organizado por tarefa; aquele, por variável.

## Valores de exemplo

Todos os endereços, IPs e identificadores nesta documentação são **fictícios**.
Troque pelos seus antes de usar:

| Neste documento | Troque por |
|---|---|
| `vault.example.com` | O domínio real do cofre |
| `example.com`, `example.net`, `example.org` | Os domínios de e-mail aceitos |
| `admin@example.com` | O e-mail do administrador |
| `no-reply@example.com` | A caixa remetente dos convites |
| `Example Corp` | O nome da organização no Vaultwarden |
| `100.64.0.10` | IP do PostgreSQL na tailnet |
| `203.0.113.10` | IP fixo do escritório, no bloqueio do `/admin` |
| `ghcr.io/<seu-usuario>/key-connector` | Seu registry |
| `00000000-0000-...` | Tenant ID e Client ID do Entra |

Os blocos `100.64.0.0/10` (faixa CGNAT do Tailscale) e `203.0.113.0/24`
(TEST-NET-3, reservado para documentação) estão corretos como exemplo — só os
endereços individuais precisam mudar.

---

## O que é o Key Connector aqui

É comum ler que login sem senha mestra só existe no Bitwarden Enterprise, via
Key Connector. Para o Vaultwarden existe um equivalente — e é ele que este
stack usa. Vale entender o que é, porque a natureza dele muda a decisão de
levar para produção:

- `acul021/key-connector` é uma **reimplementação independente em Rust**, não um
  fork do Key Connector oficial da Bitwarden. O protocolo foi deduzido a partir
  do código aberto do Vaultwarden e dos clientes Bitwarden.
- O lado servidor depende do [PR #7419](https://github.com/dani-garcia/vaultwarden/pull/7419),
  que está **aberto, não mergeado**. Por isso o compose aponta para a imagem de
  fork `ghcr.io/acul021/vaultwarden` (somente amd64).
- O serviço está na versão **0.1.0**, mantido por uma pessoa, sem auditoria
  externa. Licença AGPL-3.0.

Funciona, e é o único caminho para login sem senha mestra no Vaultwarden. Mas é
uma escolha diferente de "instalar o Vaultwarden estável" — veja a próxima seção
antes de levar para produção.

---

## O que muda no modelo de segurança

Com senha mestra, a chave que decifra o cofre nunca existe no servidor. Um
invasor que tome a VPS inteira leva dados cifrados que não consegue abrir.

Com Key Connector, a chave de cada usuário fica **guardada no serviço**, cifrada
em AES-256-GCM sob a `KC_ENCRYPTION_KEY`. Quem obtiver, ao mesmo tempo:

1. o banco SQLite do Key Connector,
2. a `KC_ENCRYPTION_KEY`, e
3. um token OIDC válido (ou controle do App Registration no Entra),

decifra os cofres. Esse trade-off é inerente ao desenho do Key Connector — vale
igual no Bitwarden oficial — e é aceito por empresas em troca da conveniência.
A diferença aqui é que a implementação tem 0.1.0 e nenhuma auditoria.

Consequências práticas de guardar em algum lugar por escrito:

- **`KC_ENCRYPTION_KEY` e o banco do KC não podem ficar no mesmo backup.** Se
  ficarem, o backup vira uma cópia em claro de todos os cofres.
- **Perder qualquer um dos dois = perda total.** Não é "usuários fazem login de
  novo": é todos os cofres permanentemente indecifráveis, sem recuperação.
- O `/keyconnector` precisa ser alcançável pelos clientes (web vault, extensão,
  app, desktop), então é um serviço exposto que guarda as chaves de todo mundo.
- **Owner e Admin da organização não podem se inscrever no Key Connector** —
  restrição do próprio PR, alinhada à documentação da Bitwarden. Eles continuam
  com senha mestra, o que dá um caminho de emergência natural.

Recomendação: rode um piloto com um departamento pequeno antes de abrir para o
escritório. A seção "Modo estável sem Key Connector" mostra a volta atrás, que é
barata enquanto pouca gente estiver inscrita.

---

## O que este projeto aproveita do padrão de infraestrutura

O documento interno de padrões de infraestrutura da empresa (não versionado aqui)
foi escrito para orientar devs criando sistemas novos. Aqui o software é pronto e
de terceiro, então o que vale é o que ele descreve sobre a **infraestrutura**,
não as regras de codificação:

- **Banco:** role e database dedicados no PostgreSQL da VPS de banco, acessado
  pelo IP `100.64.x.x` da tailnet, com `DATABASE_MAX_CONNS=10` limitando o pool.
- **Portas:** tudo publicado em `127.0.0.1`. O NPMplus segue sendo o único
  componente com portas públicas.
- **Configuração:** só por variável de ambiente, listadas explicitamente no
  `environment:`, sem `.env` versionado e sem segredo no repositório.
- **Container:** não-root, `read_only: true`, `cap_drop: ALL`,
  `no-new-privileges`, imagem com tag fixa (aqui, digest).

Não se aplicam: R2 (o Vaultwarden não fala S3), `/api/health` (o endpoint é
`/alive`), logs em JSON (o Vaultwarden só emite texto) e Dockerfile
multi-stage próprio.

Uma observação operacional que continua valendo: o Vaultwarden roda as
migrations no boot e isso não é configurável. **Nunca escale o serviço além de
1 réplica** — duas instâncias subindo juntas corrompem o schema.

---

## Pré-requisitos

- [ ] `vault.example.com` apontando para a VPS (é o único domínio necessário)
- [ ] VPS de aplicação **amd64** (a imagem de fork não tem arm64)
- [ ] Acesso à VPS de banco pela tailnet
- [ ] Permissão de Application Administrator no Entra ID
- [ ] Caixa de e-mail com SMTP AUTH para os convites

---

## Passo 1 — Banco de dados

Na VPS de PostgreSQL:

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE vaultwarden WITH LOGIN PASSWORD 'GERE_UMA_SENHA_FORTE';
CREATE DATABASE vaultwarden OWNER vaultwarden ENCODING 'UTF8';
\c vaultwarden
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT ALL ON SCHEMA public TO vaultwarden;
```

No `pg_hba.conf`, libere só a faixa da tailnet:

```
host    vaultwarden    vaultwarden    100.64.0.0/10    scram-sha-256
```

```bash
sudo systemctl reload postgresql
```

Teste a partir da VPS de aplicação:

```bash
psql "postgresql://vaultwarden:SENHA@100.64.0.10:5432/vaultwarden" -c "SELECT 1;"
```

O Key Connector **não** usa este banco: o projeto compila o `sqlx` apenas com o
backend SQLite, então ele grava em `/data/keyconnector.db`. Não há opção de
apontar para o PostgreSQL.

---

## Passo 2 — Preparar a VPS

Os containers rodam com UID fixo (1000 e 10001), então os diretórios de dados
precisam ter a posse correta. **Isso é resolvido automaticamente** pelo serviço
`init-perms` do compose, que roda como root antes dos outros, ajusta o dono e
sai. Num deploy pelo Dockhand não há passo manual aqui.

Se preferir preparar à mão — ou se já existe conteúdo com dono errado de uma
tentativa anterior, caso em que o `init-perms` não resolve porque só ajusta o
diretório raiz:

```bash
sudo install -d -o 1000 -m 700 /opt/vaultwarden/data && sudo install -d -o 10001 -m 700 /opt/vaultwarden/keyconnector
```

Para consertar conteúdo já existente:

```bash
sudo chown -R 1000 /opt/vaultwarden/data && sudo chown -R 10001 /opt/vaultwarden/keyconnector
```

Repare que o `chown` define **só o dono**, não o grupo. É deliberado: o
key-connector roda como `uid=10001 gid=999` — o `Dockerfile` fixa apenas o UID
(`useradd -r -u 10001`), e o GID sai do próximo livre do sistema. Definindo só o
dono e usando modo `700`, o acesso continua correto mesmo que um rebuild futuro
atribua outro GID.

Se você reconstruir a imagem a partir de um commit mais novo, confirme o par
antes de subir:

```bash
docker run --rm --entrypoint id ghcr.io/<seu-usuario>/key-connector:<tag>
```

### Visibilidade do pacote no registry

Pacotes novos no ghcr nascem **privados**, e uma VPS sem credencial falha com
`error from registry: denied` — mensagem que o ghcr usa tanto para "não existe"
quanto para "sem acesso", o que torna o diagnóstico confuso.

**Este stack assume o pacote público.** Depois do primeiro push, vá em
*Package settings → Change visibility → Public*. A imagem não tem segredo
embutido — é build de código AGPL público — então o custo de confidencialidade
é praticamente nulo, e a VPS não precisa de credencial nenhuma.

O motivo de não manter privado é concreto: **o GHCR não aceita fine-grained
personal access tokens.** A documentação do GitHub afirma que "GitHub Packages
only supports authentication using a personal access token (classic)", e o
suporte a fine-grained foi removido do roadmap público em 2024. Na prática,
manter o pacote privado obrigaria a colocar um **token clássico de conta
inteira** na VPS — sem como restringir a um único pacote — gravado em
`~/.docker/config.json` em base64, não criptografado, e com validade a
controlar.

Trocar "esconder qual software rodamos" por essa credencial não compensa.

Se ainda assim quiser privado, o caminho é token clássico com **apenas**
`read:packages` (nunca o mesmo usado para push), login na VPS como o usuário que
roda o compose, e a data de expiração no mesmo lembrete do client secret do
Entra:

```bash
echo '<PAT_READ_PACKAGES>' | docker login ghcr.io -u <usuario-github> --password-stdin
```

```bash
docker pull ghcr.io/<seu-usuario>/key-connector:<tag>
```

Melhor que ambos, quando houver tempo: uma organização no GitHub com conta de
serviço, para o token não ficar preso à conta pessoal de quem montou o stack.

### Digest da imagem do Vaultwarden

Resolva o digest da imagem de fork — o `:testing` é tag móvel e não serve para
produção:

```bash
docker pull ghcr.io/acul021/vaultwarden:testing && docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/acul021/vaultwarden:testing
```

Cole o resultado em `VW_IMAGE`.

---

## Passo 3 — Construir a imagem do Key Connector

O repositório não publica imagem pronta, só o `Dockerfile`. Construir a partir
de um commit fixo e publicar no registry da empresa é o caminho certo aqui — para
um serviço que guarda as chaves de todo mundo, você quer controlar a cadeia de
suprimento, não puxar tag móvel de terceiro.

```bash
git clone https://github.com/acul021/key-connector.git && cd key-connector && git rev-parse HEAD
```

Fixe o commit e construa (o build é multi-stage, runtime `debian:bookworm-slim`,
usuário não-root já configurado):

```bash
git checkout <COMMIT_SHA> && docker build -t ghcr.io/<seu-usuario>/key-connector:<COMMIT_SHA> . && docker push ghcr.io/<seu-usuario>/key-connector:<COMMIT_SHA>
```

Cole a referência em `KC_IMAGE`. Revise o diff antes de subir versão nova.

---

## Passo 4 — Gerar a chave de criptografia

```bash
openssl rand -base64 32
```

Coloque em `KC_ENCRYPTION_KEY` e **guarde uma cópia offline, fora da VPS e
separada do backup do banco do Key Connector**. Essa chave e aquele banco, juntos,
abrem todos os cofres; separados, nenhum dos dois serve para nada. É por isso
que eles não podem morar no mesmo lugar.

---

## Passo 5 — Registrar a aplicação no Entra ID

Em **portal.azure.com → Microsoft Entra ID → App registrations → New registration**:

| Campo | Valor |
|---|---|
| Name | `Vaultwarden — Cofre Corporativo` |
| Supported account types | **Single tenant** |
| Redirect URI | **Web** → `https://vault.example.com/identity/connect/oidc-signin` |

O redirect URI deriva do `DOMAIN` e precisa bater caractere a caractere — sem
barra no final, sempre `https`.

**5.1 — Identificadores.** Em *Overview*, copie **Directory (tenant) ID** e
**Application (client) ID**.

**5.2 — Segredo.** Em *Certificates & secrets → New client secret*. Copie a
coluna **Value** (não o *Secret ID*) — só aparece nesse momento. **Coloque a
data de expiração no calendário da equipe**: quando vencer, todo mundo perde o
login de uma vez, e com Key Connector não há senha mestra para contornar.

**5.3 — Permissões.** Em *API permissions* → Microsoft Graph → *Delegated*:
`openid`, `profile`, `email`, `offline_access`. Clique em **Grant admin consent**.

**5.4 — Claim de e-mail (não pule).** Em *Token configuration → Add optional
claim → ID → `email`*. O Vaultwarden identifica a conta pelo e-mail; se a claim
não vier, ou se o UPN do usuário for diferente do e-mail, o login entra em loop
de volta para a tela inicial sem mensagem de erro. Confirme também que cada
usuário tem o atributo *mail* preenchido.

**5.5 — Restringir quem entra.** Em *Enterprise applications → Vaultwarden →
Properties*, ligue **Assignment required = Yes** e atribua só os grupos de
departamento que devem ter cofre.

**5.6 — Acesso Condicional.** Exija MFA para esta aplicação. Com Key Connector o
Entra passa a ser a única barreira antes do cofre — não existe mais a senha
mestra como segundo fator implícito. Essa política deixa de ser recomendação e
vira requisito.

### Por que `SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION=true`

O Entra ID não emite a claim `email_verified`, e sem essa flag o Vaultwarden
recusa todo login. Ela vem obrigatoriamente acompanhada de
`SSO_SIGNUPS_MATCH_EMAIL=false`: com verificação desconhecida **e** associação
automática ligada, alguém que não controla um endereço poderia se vincular a uma
conta existente. As duas andam juntas — não mexa em uma sem a outra.

---

## Passo 6 — ADMIN_TOKEN

```bash
docker run --rm -it ghcr.io/acul021/vaultwarden:testing /vaultwarden hash
```

Cole **o hash** em `VW_ADMIN_TOKEN` e guarde a senha num cofre à parte. Em
arquivo `.env`, troque cada `$` do hash por `$$`; quando o Dockhand injeta como
variável de ambiente real, não escape.

Como Owner/Admin não usam Key Connector, esse token é o seu segundo caminho de
emergência, independente do Entra.

---

## Passo 7 — Deploy

### Antes de apertar o botão

O compose usa `${VAR:?mensagem}` nas variáveis obrigatórias. Se o Dockhand não
entregar alguma, o deploy **falha apontando o nome da variável** em vez de subir
um container silenciosamente mal configurado — que é exatamente o risco que o
padrão da casa alerta ("no deploy remoto ela pode chegar vazia sem erro").

Confirme antes, porque nenhuma dessas o compose resolve sozinho:

| Item | Por quê |
|---|---|
| VPS é **amd64** | A imagem de fork não tem arm64. Em arm dá `exec format error` |
| `KC_IMAGE` **existe no registry** | O repositório do Key Connector só publica `Dockerfile`. Você precisa ter feito o passo 3 |
| A VPS consegue **puxar** as duas imagens | Se o pacote no ghcr for privado, precisa de `docker login` na VPS |
| `VW_IMAGE` com **digest**, não `:testing` | Tag móvel muda debaixo de você entre deploys |
| Database e role **já criados** | O Vaultwarden roda migrations no boot; sem banco, crash-loop |
| Portas `8222` e `8281` **livres** no host | Colidem em silêncio com outro stack |

Teste a renderização das variáveis antes de subir. Este comando não sobe nada e
lista o que estiver faltando:

```bash
docker compose config --quiet
```

### Subindo

```bash
cp .env.example .env
```

Preencha o `.env` (ou cadastre no Dockhand) e homologue primeiro no Docker do
escritório.

```bash
docker compose up -d && docker compose logs -f
```

Verificações:

```bash
docker compose ps && curl -sS http://127.0.0.1:8222/alive && curl -sS http://127.0.0.1:8281/alive
```

```bash
docker exec vaultwarden env | grep -E 'SSO_ENABLED|KEY_CONNECTOR|DOMAIN'
```

O `init-perms` aparece como `Exited (0)` no `docker compose ps` — é o esperado,
ele roda uma vez e sai. Se aparecer com código diferente de zero, os dois outros
serviços não sobem; veja `docker compose logs init-perms`.

---

## Passo 8 — NPMplus

Um **Proxy Host** só — `vault.example.com`:

| Campo | Valor |
|---|---|
| Scheme / Host / Port | `http` / `127.0.0.1` / `8222` |
| Websockets Support | **ligado** (obrigatório) |
| Block Common Exploits | ligado |
| SSL | wildcard `*.example.com` + Force SSL + HTTP/2 + HSTS |

Sem websocket o `/notifications/hub` não conecta e nada sincroniza em tempo real.

Aba **Advanced**:

```nginx
client_max_body_size 525m;
proxy_read_timeout 300s;
proxy_send_timeout 300s;

# Painel administrativo só pela tailnet e pelo IP do escritório
location /admin {
    allow 100.64.0.0/10;
    allow 203.0.113.10;   # <- IP fixo do escritório
    deny all;

    proxy_pass http://127.0.0.1:8222;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}

# Key Connector. As duas barras finais são o que importa: elas fazem o
# nginx trocar o prefixo /keyconnector/ pela raiz, então o serviço recebe
# /user-keys e /alive como se estivesse sozinho no domínio.
location /keyconnector/ {
    proxy_pass http://127.0.0.1:8281/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Tire uma das duas barras e o caminho chega errado no serviço: sem a de
`proxy_pass`, o nginx repassa `/keyconnector/user-keys` inteiro e o KC responde
404.

Se o WAF do NPMplus barrar o login, a regra costuma cair em
`/identity/connect/*`. Crie a exceção nesse caminho em vez de desligar o WAF.

### Por que um caminho, e não um subdomínio

O Key Connector é um serviço HTTP separado, e **quem chama é o cliente**, não o
Vaultwarden — o servidor não faz nenhuma requisição para essa URL, só a entrega
aos clientes no campo `KeyConnectorUrl`. Por isso ela precisa ser alcançável de
fora; `127.0.0.1:8281` sozinho não serve.

Servir isso como caminho no próprio domínio funciona porque os clientes montam a
chamada por concatenação simples, sem normalizar nada
(`libs/common/src/services/api.service.ts`):

```typescript
this.httpOperations.createRequest(keyConnectorUrl + "/user-keys", { ... })
```

E o servidor só valida que o valor é uma URL com protocolo — não exige origin
puro. O CSP que ele monta usa apenas o `origin()` da URL, que aqui é o próprio
`https://vault.example.com`.

O que se ganha em relação ao subdomínio: um DNS a menos, um proxy host a menos,
e o web vault passa a chamar o Key Connector em same-origin — sem CORS e sem
entrada extra no CSP.

A regra que sobra: **`VW_KEY_CONNECTOR_URL` nunca pode terminar com barra**, ou a
concatenação gera `//user-keys`.

Se quiser reduzir a superfície e ninguém acessar o cofre fora da rede
corporativa, dá para restringir só esse caminho — mas isso quebra celular sem
VPN:

```nginx
location /keyconnector/ {
    allow 100.64.0.0/10;
    allow 203.0.113.10;
    deny all;
    # ... resto igual ao bloco acima
}
```

---

## Passo 9 — Organização e departamentos

O Vaultwarden **não** sincroniza grupos do Entra ID. O Entra decide *quem
entra*; quem vê *o quê* é configurado aqui dentro.

1. **Convide a si mesmo.** Com `SIGNUPS_ALLOWED=false` ninguém se cadastra
   sozinho — nem você. Acesse `https://vault.example.com/admin`, entre com a
   **senha** do ADMIN_TOKEN (não o hash), e use *Invite User* no seu próprio
   e-mail. Aceite o convite e entre com SSO. Use o mesmo e-mail de
   `VW_ORG_CREATION_USERS`, que é quem pode criar organizações.
2. Crie a organização com o nome **exatamente igual** a
   `VW_KEY_CONNECTOR_ORG_NAME`. É por esse nome que o servidor decide anunciar o
   Key Connector aos clientes — divergiu, ninguém se inscreve.
3. Em *Collections*, crie uma coleção por departamento: Financeiro, RH, TI,
   Comercial, Jurídico, Diretoria.
4. Em *Groups* (habilitado por `ORG_GROUPS_ENABLED=true`), crie um grupo por
   departamento com acesso apenas à sua coleção.
5. Em *Members → Invite*, convide as pessoas e coloque cada uma no seu grupo.

Com `SIGNUPS_ALLOWED=false`, ninguém entra sem convite — mesmo autenticando no
Entra com sucesso.

### Onboarding enquanto não há SMTP

Sem servidor de e-mail, o Vaultwarden **aceita os convites de organização
automaticamente**: o convidado não recebe nem precisa clicar em link, vai direto
para "aguardando confirmação".

Mas a ordem importa. Como `SSO_SIGNUPS_MATCH_EMAIL=false` está ativo — exigência
de usar `SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION=true` com o Entra —, um login SSO
**não** se associa a uma conta pré-existente com o mesmo e-mail. Criar o usuário
antes e deixá-lo logar depois não funciona.

A sequência que funciona, com `VW_SIGNUPS_ALLOWED=true`:

1. A pessoa acessa o cofre e entra com SSO. A conta é criada nesse momento, sem
   e-mail nenhum.
2. Em *Members → Invite*, convide o mesmo e-mail. Sem SMTP, entra já aceito.
3. Confirme o membro e coloque no grupo do departamento.
4. Ela recarrega e passa a ver as coleções.

Durante essa janela, só entra quem estiver na whitelist de domínios **e**
atribuído à aplicação no Entra — que é onde deve ficar o controle real de quem
tem cofre. Terminado o onboarding, volte `VW_SIGNUPS_ALLOWED` para `false`.

Se algum cadastro falhar reclamando de verificação de e-mail, use
`VW_SIGNUPS_VERIFY=false` enquanto o SMTP não existir.

Papéis: mantenha **Owner** com uma ou duas pessoas e **Admin** com quem gerencia
membros. Lembre que esses dois papéis **não** se inscrevem no Key Connector e
seguem com senha mestra; **User** e **Manager** é que ganham o login sem senha.

O PR expõe os endpoints `convert-to-key-connector` (contas que já existem com
senha mestra) e `set-key-connector-key` (contas novas), e o cliente apresenta o
fluxo depois do login SSO. Valide esse caminho no piloto antes de convidar o
escritório: é a parte mais nova do conjunto.

---

## Backup

Três coisas precisam de backup, e duas delas **não podem ficar juntas**:

| O quê | Onde | Observação |
|---|---|---|
| Banco PostgreSQL | VPS de banco | Cofres, usuários, organizações. É o backup principal |
| `/opt/vaultwarden/data` | VPS de aplicação | `rsa_key.pem` e anexos |
| `/opt/vaultwarden/keyconnector` | VPS de aplicação | Chaves dos usuários, cifradas |
| `KC_ENCRYPTION_KEY` | **cofre físico / offline** | Nunca no mesmo backup do item acima |

Confirme que o database `vaultwarden` está incluído na rotina de `pg_dump` da
VPS de banco. Backup manual:

```bash
pg_dump "postgresql://vaultwarden:SENHA@100.64.0.10:5432/vaultwarden" -Fc -f vaultwarden-$(date +%F).dump
```

O banco do Key Connector é SQLite; copie a quente com o próprio SQLite, não com
`cp` (que pode pegar o arquivo no meio de uma escrita):

```bash
docker exec key-connector sqlite3 /data/keyconnector.db ".backup /data/backup.db" 2>/dev/null || echo "sem sqlite3 na imagem - pare o container antes de copiar"
```

A imagem é enxuta e provavelmente não traz o `sqlite3`. Nesse caso, o caminho
honesto é `docker compose stop key-connector`, copiar o arquivo e subir de novo
— são poucos segundos de indisponibilidade.

**Teste a restauração em homologação antes de precisar dela.** Com Key Connector
o custo de um backup que não restaura deixou de ser "reconfigurar o servidor" e
passou a ser "perder todos os cofres".

---

## Modo estável sem Key Connector

Se o piloto não convencer, ou se o PR #7419 ficar parado, a volta é simples
**enquanto poucas contas estiverem inscritas** — quem já converteu não tem senha
mestra para voltar a usar:

1. `VW_IMAGE=vaultwarden/server:1.37.2-alpine` (upstream estável, multiarch)
2. `VW_KEY_CONNECTOR_ENABLED=false`
3. `VW_SSO_ONLY=false`, para permitir login com senha mestra
4. Remova o serviço `key-connector` do compose
5. **Antes de tudo isso**, peça a cada usuário inscrito que defina uma senha
   mestra pelo cliente. Sem esse passo, a conta fica inacessível.

O SSO com Entra ID continua funcionando normalmente no upstream — o que se perde
é só o "sem senha mestra".

---

## Troubleshooting

| Sintoma | Causa provável |
|---|---|
| Login volta para a tela inicial, sem erro | Claim `email` ausente ou UPN diferente do e-mail (passo 5.4) |
| `Invalid redirect URI` | `VW_DOMAIN` diferente do registrado no Entra — confira barra final e `https` |
| Todo login recusado logo após o SSO | `SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION` desligado |
| `AADSTS7000215: Invalid client secret` | Foi copiado o *Secret ID* em vez do *Value*, ou expirou |
| Cliente não oferece o Key Connector | `KEY_CONNECTOR_ORG_NAME` diferente do nome real da organização; ou o usuário é Owner/Admin, que não podem se inscrever |
| Login parece funcionar mas volta pro início logo depois, sem erro nos logs do Vaultwarden | KC não conseguiu buscar o JWKS. Veja o item de hairpin abaixo — é a causa mais provável quando o Vaultwarden só mostra `200 OK` em tudo |
| KC responde 401 em `/user-keys` | Mesma causa do item acima |
| 404 em `/keyconnector/user-keys` | Falta a barra final no `proxy_pass` do NPMplus — o prefixo não está sendo removido |
| Requisição vai para `//user-keys` | `VW_KEY_CONNECTOR_URL` terminou com barra |
| KC não sobe, erro de permissão em `/data` | Diretório do host não é do UID 10001 |
| Vaultwarden reinicia, erro de permissão em `/data` | Diretório do host não é do UID 1000 |
| `exec format error` no vaultwarden | VPS arm64 — a imagem de fork é só amd64 |
| Cofre não sincroniza entre dispositivos | Websockets desligado no proxy host do `vault.example.com` |
| Convites não chegam | SMTP AUTH bloqueado no Microsoft 365 |

**Hairpin.** O Key Connector chama `KC_IDENTITY_AUTHORITY`, que é o próprio
domínio público (`https://<dominio>/identity`) — sairia do container, iria até
o IP público da VPS e voltaria pelo NPMplus. Muitos provedores não suportam essa
volta ("hairpin NAT"), e a busca do JWKS falha de um jeito que não aparece nos
logs do Vaultwarden: o login autentica normal, mas o navegador não consegue
completar a inscrição no Key Connector e volta para o início sem erro visível.

O compose já vem com a correção — `extra_hosts: ["${VW_DOMAIN_HOST}:host-gateway"]`
no serviço `key-connector` — que resolve o domínio para o próprio host dentro do
container, sem sair da VPS. Depende de `VW_DOMAIN_HOST` estar preenchido com
**apenas o hostname**, sem `https://` nem barra — confira se não ficou esquecido
no `.env`, é fácil passar batido porque o compose não consegue derivá-lo
sozinho a partir de `VW_DOMAIN`.

Se mesmo assim o problema persistir — provedor com rede exótica onde nem
`host-gateway` resolve — a alternativa é abandonar o discovery e fixar a chave
pública, exportando-a do Vaultwarden:

```bash
openssl rsa -in /opt/vaultwarden/data/rsa_key.pem -pubout -out /opt/vaultwarden/keyconnector/identity.pub.pem
```

Aí troque `KC_IDENTITY_AUTHORITY` por `KC_IDENTITY_PUBLIC_KEY_PATH` +
`KC_JWT_ISSUER`. Custo: precisa reexportar se o `rsa_key.pem` mudar.

Diagnóstico de SSO: `SSO_DEBUG_TOKENS=true` temporariamente. Ele **loga os
tokens** — desligue em seguida e limpe os logs.

---

## Referências

- [acul021/key-connector](https://github.com/acul021/key-connector)
- [Vaultwarden PR #7419 — Key Connector support](https://github.com/dani-garcia/vaultwarden/pull/7419)
- [Vaultwarden Wiki — SSO com OpenID Connect](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect)
- [Vaultwarden — releases](https://github.com/dani-garcia/vaultwarden/releases)
- [Bitwarden — About Key Connector](https://bitwarden.com/help/about-key-connector/)
- [Discussão: Entra ID com UPN diferente do e-mail](https://github.com/dani-garcia/vaultwarden/discussions/7152)
