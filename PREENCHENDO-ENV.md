# Como preencher o `.env`

O README está organizado por tarefa. Este arquivo está organizado por
**variável**, na ordem em que dá para conseguir cada valor. Siga de cima para
baixo: cada etapa só depende das anteriores.

Marcações: **[decidir]** você escolhe · **[gerar]** sai de um comando ·
**[buscar]** existe em outro sistema.

---

## Etapa A — Decisões (5 minutos, nada a instalar)

Comece por aqui: quatro variáveis derivam de uma única escolha.

### `VW_DOMAIN` **[decidir]**

O endereço público do cofre. Precisa existir no DNS apontando para a VPS e estar
coberto pelo certificado wildcard `*.example.com`.

```
VW_DOMAIN=https://vault.example.com
```

Regras: com `https://`, **sem barra no final**. Se você usar outro nome
(`vault.example.com`, `senhas.example.com`), troque aqui e nas duas de baixo.

### `VW_KEY_CONNECTOR_URL` e `KC_IDENTITY_AUTHORITY` **[derivadas]**

Não são escolhas independentes — saem do `VW_DOMAIN`:

```
VW_KEY_CONNECTOR_URL=https://vault.example.com/keyconnector
KC_IDENTITY_AUTHORITY=https://vault.example.com/identity
```

Trocou o domínio, troque as duas junto. A primeira **nunca** termina com barra.

### `VW_KEY_CONNECTOR_ORG_NAME` e `VW_INVITATION_ORG_NAME` **[decidir]**

O nome da organização que você vai criar dentro do Vaultwarden depois do deploy.

```
VW_KEY_CONNECTOR_ORG_NAME=Example Corp
VW_INVITATION_ORG_NAME=Example Corp
```

O primeiro precisa ser **idêntico** ao nome que você digitar na hora de criar a
organização — é por ele que o servidor decide anunciar o Key Connector aos
clientes. Divergiu no acento ou no espaço, ninguém se inscreve.

### `VW_SIGNUPS_DOMAINS_WHITELIST` **[decidir]**

Domínios de e-mail aceitos. Separe por vírgula se houver mais de um.

```
VW_SIGNUPS_DOMAINS_WHITELIST=example.com
```

### `VW_ORG_CREATION_USERS` **[decidir]**

Quem pode criar organizações. Coloque o seu e-mail — você vai criar a
organização no pós-deploy.

```
VW_ORG_CREATION_USERS=admin@example.com
```

### `VW_KEY_CONNECTOR_ENABLED` e `VW_SSO_ONLY` **[decidir]**

Deixe as duas como `true`. É a combinação correta para uma instalação nova com
Key Connector: os usuários não terão senha mestra, então o login por e-mail+senha
precisa estar fechado.

```
VW_KEY_CONNECTOR_ENABLED=true
VW_SSO_ONLY=true
```

Se você preferir subir primeiro o Vaultwarden estável e só depois ligar o Key
Connector, é aqui que se muda — junto com `VW_IMAGE`. O README documenta esse
caminho em "Modo estável sem Key Connector".

---

## Etapa B — Valores que você gera (2 minutos)

### `KC_ENCRYPTION_KEY` **[gerar]**

```bash
openssl rand -base64 32
```

Cole a saída inteira, incluindo o `=` do final.

**Antes de seguir:** guarde uma cópia dessa chave fora da VPS — cofre físico,
gerenciador de senhas pessoal, o que for. Ela e o banco do Key Connector, juntos,
abrem todos os cofres; separados, nenhum dos dois serve. Perder os dois é perda
total e sem recuperação. É o único valor deste arquivo com essa característica.

### `VW_ADMIN_TOKEN` **[gerar]**

Não é uma senha em texto puro — é o hash dela:

```bash
docker run --rm -it ghcr.io/acul021/vaultwarden:testing /vaultwarden hash
```

O comando pede uma senha, você digita, e ele devolve um hash começando com
`$argon2id$v=19$...`. **Guarde a senha que você digitou** (é ela que você usa
para entrar em `/admin`) e cole **o hash** na variável.

**O detalhe que quebra:** em arquivo `.env`, troque cada `$` do hash por `$$`:

```
VW_ADMIN_TOKEN=$$argon2id$$v=19$$m=65540,t=3,p=4$$c29tZXNhbHQ$$hash...
```

Se o Dockhand injetar como variável de ambiente real, **não** escape — cole o
hash original. Errar isso é a causa mais provável de "a senha do /admin não
funciona".

---

## Etapa C — Banco de dados

### `VW_DATABASE_URL` **[gerar + buscar]**

Monte a partir de quatro pedaços:

```
postgresql://<usuario>:<senha>@<ip-tailnet>:5432/<database>
```

1. **Senha** — você gera: `openssl rand -base64 24`
2. **IP da tailnet** — o da VPS de PostgreSQL, formato `100.64.x.x`.
   Descubra com `tailscale status` na VPS de banco.
3. **Usuário e database** — você cria, seguindo o passo 1 do README:

```sql
CREATE ROLE vaultwarden WITH LOGIN PASSWORD 'a-senha-que-voce-gerou';
CREATE DATABASE vaultwarden OWNER vaultwarden ENCODING 'UTF8';
```

Resultado:

```
VW_DATABASE_URL=postgresql://vaultwarden:SENHA_GERADA@100.64.0.10:5432/vaultwarden
```

Teste antes de seguir — se isso não responder, o container entra em crash-loop:

```bash
psql "postgresql://vaultwarden:SENHA@100.64.0.10:5432/vaultwarden" -c "SELECT 1;"
```

Se a senha tiver `$`, escape como `$$` no `.env` (mesma regra do ADMIN_TOKEN).
Mais simples: gere uma senha sem `$`.

---

## Etapa D — Microsoft Entra ID

Precisa de alguém com permissão de **Application Administrator** ou Global Admin
no tenant. Se não for você, é aqui que você para e pede.

Siga o passo 5 do README para criar o App Registration. Ao final, três valores:

### `VW_SSO_AUTHORITY` **[buscar]**

Em *Entra ID → App registrations → sua app → Overview*, copie o
**Directory (tenant) ID** e monte:

```
VW_SSO_AUTHORITY=https://login.microsoftonline.com/<TENANT_ID>/v2.0
```

O `/v2.0` no final é obrigatório. Sem ele, o Vaultwarden não acha o discovery.

### `VW_SSO_CLIENT_ID` **[buscar]**

Na mesma tela, o **Application (client) ID**. É um GUID:

```
VW_SSO_CLIENT_ID=a1b2c3d4-1234-5678-9abc-def012345678
```

### `VW_SSO_CLIENT_SECRET` **[buscar]**

Em *Certificates & secrets → New client secret*. A tabela mostra duas colunas —
copie a **Value**, não a *Secret ID*. Ela some assim que você sair da página.

```
VW_SSO_CLIENT_SECRET=abc7Q~xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Anote a data de expiração no calendário da equipe. Quando vencer, **todo mundo
perde o login de uma vez** — e com Key Connector não há senha mestra para
contornar.

---

## Etapa E — Imagens

### `VW_IMAGE` **[gerar]**

Resolva o digest da imagem de fork:

```bash
docker pull ghcr.io/acul021/vaultwarden:testing
```

```bash
docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/acul021/vaultwarden:testing
```

Cole a saída completa:

```
VW_IMAGE=ghcr.io/acul021/vaultwarden@sha256:abcdef0123...
```

Por que não usar `:testing` direto: é tag móvel. O mantenedor publica de novo, e
seu próximo deploy sobe código diferente sem você saber.

### `KC_IMAGE` **[gerar]**

Esta é a única que exige trabalho de verdade — o repositório do Key Connector
publica só o `Dockerfile`, não uma imagem pronta.

```bash
git clone https://github.com/acul021/key-connector.git && cd key-connector && git rev-parse HEAD
```

Anote o commit e construa:

```bash
docker build -t ghcr.io/<seu-usuario>/key-connector:<COMMIT_SHA> . && docker push ghcr.io/<seu-usuario>/key-connector:<COMMIT_SHA>
```

```
KC_IMAGE=ghcr.io/<seu-usuario>/key-connector:a1b2c3d
```

Se a organização `example` não existir no GitHub, use sua própria conta
(`ghcr.io/<seu-usuario>/key-connector:<sha>`). Nos dois casos, se o pacote ficar
privado, a VPS precisa de `docker login ghcr.io` antes do deploy — senão o pull
falha.

---

## Etapa F — SMTP (pode ficar para depois)

> **O Vaultwarden não faz OAuth para SMTP.** Não existe caminho para entregar
> app id + client secret direto ao Microsoft 365: a biblioteca até suporta
> XOAUTH2, mas o Vaultwarden não obtém nem renova o token, e ele expira em uma
> hora. Se você quer credencial de aplicativo em vez de senha de pessoa, o
> caminho é o **Azure Communication Services** — ver abaixo.
>
> Todo este bloco é **opcional no compose**: `VW_SMTP_HOST` vazio desliga o
> envio e o servidor sobe normalmente. Dá para implantar e testar SSO e Key
> Connector antes do relay existir. Para criar a primeira conta sem e-mail, veja
> "Bootstrap sem SMTP" no fim deste arquivo.

### Opção A — Azure Communication Services (credencial de aplicativo)

```
VW_SMTP_HOST=smtp.azurecomm.net
VW_SMTP_PORT=587
VW_SMTP_SECURITY=starttls
VW_SMTP_USERNAME=<recurso-ACS>.<AppId-do-app-de-email>.<TenantId>
VW_SMTP_PASSWORD=<client secret do app de e-mail>
```

Passos no Azure:

1. Criar o recurso **Communication Services**. Anote o nome.
2. Em *Domains*, configurar o domínio remetente. Domínio próprio exige
   verificação DNS + SPF + DKIM; o gerenciado pelo Azure funciona na hora, mas
   o convite sai de um endereço `@*.azurecomm.net`.
3. Criar um **App Registration dedicado** para envio — não reaproveite o do SSO.
   São ciclos de vida diferentes, e rotacionar o segredo do e-mail derrubaria o
   login de todos se fossem o mesmo app.
4. No recurso ACS → *Access control (IAM)*, atribuir ao app a role de envio.

O usuário concatenado fica longo e há relatos de problema com limite de tamanho
do campo — teste um envio antes de considerar fechado.

### Opção B — Microsoft 365 direto

```
VW_SMTP_HOST=smtp.office365.com
VW_SMTP_USERNAME=<caixa>@example.com
VW_SMTP_PASSWORD=<senha da caixa>
```

Exige **mailbox licenciada** — caixa compartilhada não tem senha e não faz SMTP
AUTH, que é o erro clássico aqui. E alguém precisa habilitar *Authenticated
SMTP* na caixa (Exchange admin center → *Manage email apps*), que vem desativado
por padrão.

### Referência antiga

Usado para os convites. Sem isso você não consegue convidar ninguém.

```
VW_SMTP_HOST=smtp.office365.com
VW_SMTP_FROM=no-reply@example.com
VW_SMTP_USERNAME=no-reply@example.com
VW_SMTP_PASSWORD=<senha da caixa>
```

**O obstáculo previsível:** o Microsoft 365 desativou SMTP AUTH básico por
padrão. Você vai precisar de uma destas:

- habilitar SMTP AUTH especificamente na caixa de serviço (Exchange admin
  center → a caixa → *Manage email apps* → *Authenticated SMTP*), ou
- usar um relay transacional (Resend, Amazon SES, Brevo) — nesse caso troque
  `VW_SMTP_HOST` e credenciais pelos do provedor.

Se a senha tiver `$`, escape como `$$`.

`VW_SMTP_PORT`, `VW_SMTP_SECURITY` e `VW_SMTP_FROM_NAME` têm default no compose
e podem ficar como estão.

---

## Etapa G — Pode deixar como está

Estas têm default e não bloqueiam o deploy:

```
VW_DATABASE_MAX_CONNS=10
VW_SSO_MASTER_PASSWORD_POLICY={"minComplexity":3,...}
VW_PUSH_ENABLED=false
VW_PUSH_INSTALLATION_ID=
VW_PUSH_INSTALLATION_KEY=
```

`VW_PUSH_*` só importa se quiser notificação push nos apps móveis — exige
registro em https://bitwarden.com/host/ e pode ficar para depois.

---

## Conferindo antes de subir

```bash
docker compose config --quiet
```

Silêncio = tudo preenchido. Qualquer variável faltando aparece pelo nome.

Isso valida o preenchimento, **não** os valores: uma senha de banco errada ou um
client secret trocado passam por aqui e só falham no boot. Por isso o teste do
`psql` na etapa C.

---

## Depois do primeiro deploy: criando sua conta

Um detalhe que trava muita gente. O compose sobe com `SIGNUPS_ALLOWED=false`,
então **você não consegue se cadastrar sozinho** — nem sendo o administrador.

O caminho é pelo painel administrativo:

1. Acesse `https://vault.example.com/admin` e entre com a **senha** do ADMIN_TOKEN
   (a que você digitou na etapa B, não o hash).
2. Em *Invite User*, convide o seu próprio e-mail.
3. Aceite o convite que chegar por e-mail e entre com SSO.
4. Crie a organização com o nome exato de `VW_KEY_CONNECTOR_ORG_NAME`.
5. Só então siga o passo 9 do README para as coleções e grupos por departamento.

Se o convite não chegar, o problema é SMTP (etapa F), não o cofre.

### Bootstrap sem SMTP

Se o relay ainda não estiver pronto, o convite do passo 2 não chega e você fica
travado. Saída: deixe o SSO criar a conta sozinho.

1. Suba com `VW_SIGNUPS_ALLOWED=true`.
2. Acesse `https://vault.example.com` e entre com SSO. A conta é criada no primeiro
   login, sem e-mail nenhum.
3. Crie a organização.
4. **Volte para `VW_SIGNUPS_ALLOWED=false` e refaça o deploy.**

O risco durante essa janela é limitado: mesmo com o auto-cadastro ligado, só
entra quem estiver na whitelist de domínio **e** atribuído à aplicação no Entra
(`Assignment required = Yes`). Ainda assim, é uma janela — feche assim que criar
sua conta, e faça isso antes de convidar qualquer outra pessoa.
