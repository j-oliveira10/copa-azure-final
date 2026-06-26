# Guia do Aluno — Quartas de Final (F2: Gateway YARP + Identidade dois-mundos) do zero

> **O que você vai construir nesta aula:** a **Fase 2 (F2)** do FIFA 2026 Tickets — colocar **identidade** na frente do sistema. Duas pessoas diferentes entram no app, por **dois mundos diferentes**:
> - O **cliente** (compra ingresso) entra pelo **Entra External ID / CIAM** (`<tenant>.ciamlogin.com`) — cadastro self-service, login com Google ou código por email.
> - O **admin** (operador) entra pelo **workforce** (`login.microsoftonline.com`) — conta corporativa, com a App Role `Admin`.
>
> Você sobe o **gateway YARP** (a porta da frente), faz ele validar os **dois** tipos de login com a mesma mecânica, e no final **migra** usuários antigos (senha bcrypt) para o CIAM **sem apagar nada** — provando que as duas identidades convivem na mesma linha do banco.
>
> **Importante (leia antes de começar):**
> - **Cada aluno cria TUDO no próprio ambiente** (seu fork, sua subscription, seus nomes). Não há recurso compartilhado.
> - **Esta aula é cumulativa:** você reaproveita sua **Function App F1**, seu **SQL** e seu **Web App de frontend** das Oitavas. A F2 entra **na frente** disso, sem reescrever a Function.
> - **CÓDIGO** (gateway, frontend, migrations) sai pelo **GitHub Actions** do fork (workflow `Lab Quartas de Final`). **AMBIENTE e IDENTIDADE** (ACR, Container App, tenant CIAM, App Registrations, App Settings) você faz **à mão** no Portal/Entra admin center.
> - **Teoria completa (leia antes):** [`README.md`](../workshops/phase-03-quartas/README.md) (35 min). Decisões: [ADE-007](../architecture/ade-007-identity-external-id.md).

---

## Antes de tudo — os 3 conceitos que você PRECISA entender (5 min)

1. **Dois mundos de identidade, dois produtos, duas URLs de login:**

   | | **Cliente (comprador)** | **Admin (operador)** |
   |---|---|---|
   | Produto | **Entra External ID** (CIAM) | **Entra ID** (workforce) |
   | URL de login | **`<tenant>.ciamlogin.com`** | **`login.microsoftonline.com`** |
   | Como entra | cadastro self-service (Google / email+OTP) | conta corporativa que já existe |

   > 🧠 **Analogia do estádio:** o cliente compra na **bilheteria pública** (qualquer um chega, se cadastra) = External ID. O funcionário entra pela **portaria de serviço** com o **crachá** da empresa = workforce.
   >
   > ⚠️ **Erro nº1 do lab:** confundir os domínios. Login do cliente em `login.microsoftonline.com` (em vez de `ciamlogin.com`) → `AADSTS50011`. E **nunca** use `b2clogin.com` (Azure AD B2C **legado** — não criamos nada de B2C).

2. **O gateway é "issuer-agnóstico":** valida qualquer token **por discovery** (busca a chave pública do emissor numa URL `.well-known`). Aceitar um novo mundo de identidade é **configuração** (mudar a string da authority), não código novo.

3. **A migração é ADITIVA:** ao "migrar" um usuário antigo pro CIAM, a senha bcrypt dele **fica intacta**. Você só **adiciona** um ponteiro (`users.entra_oid`). O mesmo humano passa a ter duas credenciais independentes — e provar isso é o clímax do lab.

---

## Parte 0 — Visão geral: o que é "à mão" e o que é "Actions"

```
              VOCÊ (Portal Azure + Entra admin center + Google)        SEU FORK (GitHub Actions)
              ──────────────────────────────────────────────          ─────────────────────────
  ┌────────────────────────────────────────────────────────┐
  │  Conta Google → OAuth client + Gemini key                │
  │  Tenant CIAM (External ID) + user flow + Google/OTP      │
  │  App Reg SPA (cliente) · App Reg admin (workforce)       │ ──┐  (você cria/configura à mão)
  │  ACR + Container App do gateway + App Settings (Jwt__*)  │   │
  └────────────────────────────────────────────────────────┘   │
                                                                 ▼
                            ┌──────────────────────────────────────────────────────┐
                            │  Workflow ÚNICO "Lab Quartas de Final", input `acao`:  │
                            │    migrations → phase-01 + phase-03 + phase-04-ciam    │
                            │    gateway    → build/test/push imagem + deploy CA     │
                            │    frontend   → build Vite (VITE_CIAM_*) + deploy      │
                            └──────────────────────────────────────────────────────┘
                                                                 │
                                                                 ▼
   Runtime:  Browser loga no CIAM (MSAL/PKCE) → token (JWT ciamlogin.com)
             → POST /purchase com Authorization: Bearer <token>
             → Gateway valida o JWT, extrai `oid`, injeta X-Entra-OID
             → Function F1 (INALTERADA) grava purchases.entra_oid
```

**Regra de ouro:** o Portal/Entra/Google criam e configuram a identidade; os Actions só publicam **código** (gateway, frontend) e **schema** (migrations).

---

## Convenção de nomes (preencha a SUA)

Escolha um prefixo pessoal (ex.: suas iniciais `jds`) e 3 dígitos `<rand>`. **Reaproveite** os recursos da F1; crie só o que é **NOVO** da F2.

| Recurso | Padrão / Placeholder | Seu valor | Observação |
|---|---|---|---|
| Resource Group | `<seu-rg>` | ________ | **reusa** o da F1 |
| Região | `<sua-regiao>` | ________ | **mesma** da F1 |
| SQL Server / DB | `<seu-sql-server>` / `FIFA2026Tickets` | ________ | **reusa** da F1 |
| Function App F1 | `<seu-func>` | ________ | **reusa** da F1 |
| Web App frontend | `<seu-frontend>` | ________ | **reusa** da F1 |
| Conta Google do lab | `fifa2026.lab.<algo>@gmail.com` | ________ | **NOVA** (Fase 0) |
| Container Registry (ACR) | `acrfifa2026<iniciais><rand>` | ________ | **NOVO** — só letras/números |
| Container Apps Environment | `cae-fifa2026-<iniciais>` | ________ | **NOVO** |
| Container App (gateway) | `gateway-<iniciais>` | ________ | **NOVO** |
| Tenant CIAM (External ID) | `<tenant>` (`<tenant>.ciamlogin.com`) | ________ | **NOVO** — trial 30d (Fase 2) |
| App Reg SPA (cliente) | `student-<iniciais>-v2` | ________ | **NOVO** — no tenant CIAM |
| App Reg admin | `student-<iniciais>-admin` | ________ | **NOVO** — no workforce |

> 💡 Você terá **DOIS** tenants e **DOIS** client IDs: o **CIAM** (cliente, `*.ciamlogin.com`) e o **workforce** (admin). Anote os dois Tenant IDs e os dois Client IDs separadamente — é fácil confundir.

---

## Referência rápida — Variáveis & Secrets do GitHub Actions

> Fork → **Settings → Secrets and variables → Actions**. Você preenche ao longo das fases.

### 🔑 Secrets

| Secret | Bloco | De onde vem | Fase |
|---|---|---|---|
| `AZURE_CREDENTIALS` | migrations + gateway | JSON do Service Principal (o mesmo das Oitavas serve) | 1 |
| `SQL_CONNECTION_STRING` | migrations | connection string do banco (Cloud Shell PowerShell) | 1 |
| `AZURE_FRONTEND_PUBLISH_PROFILE` | frontend | publish profile do Web App (capture **após** ligar o SCM basic-auth) | 5 |

### 📋 Variables

| Variable | Bloco | Valor (o SEU) | Default | Fase |
|---|---|---|---|---|
| `SQL_SERVER` | migrations | `<seu-sql-server>` | `sql-dev-tk-cin-001` | 1 |
| `RESOURCE_GROUP` | migrations | `<seu-rg>` | `rg-hml-tik-cin-001` | 1 |
| `ACR_LOGIN_SERVER` | gateway | `<seu-acr>.azurecr.io` | — | 4 |
| `PHASE04_CONTAINERAPP_NAME` | gateway | `gateway-<iniciais>` | — | 4 |
| `PHASE04_RESOURCE_GROUP` | gateway | `<seu-rg>` | — | 4 |
| `FRONTEND_APP_NAME` | frontend | `<seu-frontend>` | — | 5 |
| `VITE_CIAM_AUTHORITY` | frontend | `https://<tenant>.ciamlogin.com/` | — | 5 |
| `VITE_CIAM_CLIENT_ID` | frontend | Client ID da App Reg SPA | — | 3 |
| `GATEWAY_V2_URL` | frontend | URL pública do gateway | — | 4 |
| `BACKEND_URL` | frontend | URL do backend v1 (proxy do `web.config`) | — | 5 |
| `FUNCTION_V2_URL` | frontend | `https://<seu-func>.azurewebsites.net` | — | 5 |

> ⚠️ **Débito de naming:** as vars do gateway ainda têm o prefixo **`PHASE04_`** — configure exatamente assim (é o que o workflow lê).

---

## Fase 0 — Conta Google nova do lab (credenciais reaproveitáveis)

> Crie uma conta **Gmail nova e exclusiva** do lab (ex.: `fifa2026.lab.<iniciais>@gmail.com`). Ela gera **duas** credenciais que você reusa: o **OAuth Client** (login Google do CIAM, Fase 2) e a **API key do Gemini** (para o **último lab**, o chatbot). Logue nela em janela anônima/perfil separado.

### 0.1 Gerar a API key do Gemini (guarde para o último lab)

> ⚠️ Hoje a key se cria no **Google AI Studio** (não no Cloud Console).

1. Acesse **https://aistudio.google.com/apikey** logado na conta do lab → aceite os termos.
2. **Create API key** → **Create API key in new project**.
3. **Copie a chave imediatamente** e guarde como `GEMINI_API_KEY` (nunca no código).

> Isto **não** é usado nas Quartas — é um adiantamento para o lab do chatbot LLM (`gemini-2.5-flash`). Como você já está na conta, deixa pronto.

*(O OAuth Client do Google é criado na Fase 2, porque depende do Tenant ID do CIAM. Passo a passo detalhado no **Apêndice A**.)*

✅ **Checkpoint:** conta Gmail do lab criada; Gemini key guardada.

---

## Fase 1 — Migrations do banco (workflow `acao=migrations`)

> **Das Oitavas você já rodou** `phase-01` (source/correlation_id) e `phase-03` (`purchases.entra_oid`). A **única migration NOVA das Quartas** é **`phase-04-ciam-link.sql`**, que cria a coluna `users.entra_oid` **vazia** (o preenchimento é hands-on na Fase 8). O workflow repete as três por serem idempotentes — não tem efeito colateral.

1. No fork, confirme os Secrets `AZURE_CREDENTIALS` + `SQL_CONNECTION_STRING` e as Variables `SQL_SERVER` + `RESOURCE_GROUP`.

   > **Montar a `SQL_CONNECTION_STRING` (Cloud Shell PowerShell):**
   > ```powershell
   > $server = "<seu-sql-server>"; $senha = "<senha-adminsql>"
   > "Server=$server.database.windows.net,1433;Database=FIFA2026Tickets;User Id=adminsql;Password=$senha;Encrypt=true;TrustServerCertificate=true"
   > ```

2. **Actions → "Lab Quartas de Final" → Run workflow** → `acao = migrations` → branch `phase-04-quartas`.

✅ **Checkpoint:** workflow verde; a coluna `users.entra_oid` existe (vazia) + índice `UQ_users_entra_oid`.

---

## Fase 2 — Tenant CIAM (Entra External ID) + user flow + Google IdP

> Tudo aqui é no **Microsoft Entra admin center** ([entra.microsoft.com](https://entra.microsoft.com)) — **não** no `portal.azure.com`.

### 2.1 Criar o tenant External ID (trial 30 dias, sem subscription)

1. Acesse [entra.microsoft.com](https://entra.microsoft.com). ⚠️ Use uma **conta Microsoft que ainda não criou um tenant External ID** — o trial gratuito só é oferecido na **primeira** criação.
2. **Entra ID → Overview → Manage tenants → `Create`**.
3. Selecione **External** → **Continue**.
4. ⚠️ Escolha a opção **30-day free trial** (a doc afirma: *"an Azure subscription isn't required"*). **NÃO** escolha "Use Azure Subscription" (essa exige subscription + RG). Não pede cartão.
5. Informe **Tenant Name** e **Domain Name**, escolha **Country/Region** (⚠️ a região **não muda depois**).
6. A criação leva **até 30 min** (acompanhe em **Notifications**).
7. Troque para o tenant novo: engrenagem (**Settings**) → **Directories + subscriptions → Switch**. Em **Tenant overview**, anote **Tenant ID** e **Primary domain** (`<tenant>.ciamlogin.com`).

✅ **Checkpoint:** tenant CIAM criado; domínio `*.ciamlogin.com` confirmado; **Tenant ID** anotado.

### 2.2 Criar o user flow (sign-up/sign-in)

1. No tenant CIAM: **Entra ID → External Identities → User flows → `New user flow`**.
2. **Name** (ex.: `SignUpSignIn`).
3. **Identity providers:** marque **Email Accounts** → escolha **Email one-time passcode** (login por código — fallback de zero dependência) **ou** Email with password.
   > ⚠️ A opção **Google** só aparece aqui **depois** de configurar a federação (2.3).
4. **User attributes:** escolha os atributos a coletar (email, display name…). **Create**.

### 2.3 Adicionar o Google como Identity Provider

> Pré-requisito: o **OAuth Client do Google** (Client ID + secret) — crie-o agora seguindo o **Apêndice A** (precisa do Tenant ID da 2.1 para os redirect URIs). Volte aqui com Client ID + secret em mãos.

1. No tenant CIAM: **Entra ID → External Identities → All identity providers**.
2. Aba **Built-in** → ao lado de **Google** → **Configure**.
3. Preencha **Name** (`Google`), **Client ID** e **Client secret** (do Apêndice A) → **Save**.
4. Adicione ao user flow: **External Identities → User flows →** o flow **→ Settings → Identity providers →** marque **Google → Save**.

✅ **Checkpoint:** Google federado e disponível no user flow (com email+OTP de fallback).

---

## Fase 3 — App Registration SPA (cliente) no tenant CIAM

1. No tenant CIAM: **Entra ID → App registrations → `New registration`**.
2. **Name:** `student-<iniciais>-v2` · **Account types:** **Accounts in this organizational directory only** (single-tenant).
3. **Register** → na **Overview**, anote **Application (client) ID** e **Directory (tenant) ID**.
4. **Authentication → Add a platform → Single-page application** → **Redirect URI:** `http://localhost:5173` (dev Vite) e a URL de prod do frontend. **NÃO** crie client secret (SPA é público, usa PKCE).
   > ⚠️ Use a plataforma **Single-page application** (marca o redirect como tipo `spa` automaticamente). Se escolher "Web" por engano, corrija o tipo no **Manifest**.
5. Vincule ao user flow: **External Identities → User flows →** o flow **→ Use → Applications → `Add application`** → selecione `student-<iniciais>-v2` → **Select**.
   > ⚠️ Há um app `b2c-extensions-app` na lista — **não apague** (guarda atributos de extensão).

✅ **Checkpoint:** App Reg SPA no CIAM; **client ID** anotado; redirect SPA + vínculo ao user flow. Guarde o **client ID** para `VITE_CIAM_CLIENT_ID` (Fase 5).

> ⚠️ **Armadilha:** criar a app no tenant **workforce** por engano. Confirme `*.ciamlogin.com` no topo do Entra admin center.

---

## Fase 4 — Gateway YARP: ACR + Container App (Bloco 1)

> O gateway é a porta da frente. ⚠️ **Importante:** o gateway das Quartas é **fail-closed** — ele **não sobe** sem `Jwt__CiamTenantId`. Por isso o tenant CIAM (Fase 2) já está pronto **antes** desta fase.

### 4.1 Criar o Container Registry (ACR)

1. Portal → **"Container registries" → `+ Create`**.
2. **RG:** `<seu-rg>` · **Registry name:** `acrfifa2026<iniciais><rand>` (⚠️ **só letras e números**) · **Location:** `<sua-regiao>` · **Pricing:** **Basic**.
3. **Review + create → Create → Go to resource**. Anote o **Login server** (`...azurecr.io`).
4. **Settings → Access keys →** habilite **Admin user** (anote username/password — o Container App usa para puxar a imagem).

### 4.2 Publicar a imagem do gateway (Cloud Shell)

> Pela primeira vez, à mão, para entender as peças (depois o workflow faz isso).

1. Portal → ícone **Cloud Shell** (`>_`) → **Bash**.
2. Build remoto no ACR (não precisa de Docker local; o `Dockerfile` está em `src/Fifa2026.V2.Gateway/`, expõe a porta **8080**):
   ```bash
   az acr build --registry acrfifa2026<iniciais><rand> --image gateway:v1 src/Fifa2026.V2.Gateway
   ```

### 4.3 Criar o Container App do gateway

1. Portal → **"Container Apps" → `+ Create`**.
2. **Basics:** RG `<seu-rg>` · **Name:** `gateway-<iniciais>` · **Region:** `<sua-regiao>` · **Container Apps Environment:** `Create new` → `cae-fifa2026-<iniciais>`.
3. **Container:** desmarque "Use quickstart image" → **Image source:** Azure Container Registry → **Registry:** seu ACR → **Image:** `gateway` → **Tag:** `v1`.
4. **Ingress:** **Enabled** · **Accepting traffic from anywhere** · **Target port:** **`8080`** ⚠️ (errar aqui = 502).
5. **Review + create → Create → Go to resource**. Anote a **Application Url** → vira `GATEWAY_V2_URL` (Fase 5).

### 4.4 Configurar as App Settings de identidade (o gateway só sobe com elas)

No Container App → **Containers → Edit and deploy → Environment variables**:

| App Setting | Valor | O que faz |
|---|---|---|
| `Jwt__CiamTenantId` | **Tenant ID do CIAM** (Fase 2.1) | authority/issuer CIAM |
| `Jwt__CiamClientId` | **Client ID da App Reg SPA** (Fase 3) | `aud` esperado do token |
| `FunctionAppF1Url` | `https://<seu-func>.azurewebsites.net` | para onde o gateway encaminha (sem ela → 502) |
| `Gateway__FrontendOrigin` | `https://<seu-frontend>...` | CORS restrito ao front |

> 🔒 **Duplo underscore:** `Jwt:CiamTenantId` em variável de ambiente vira `Jwt__CiamTenantId`. A connection string do SQL **NÃO** vai no gateway (fica na Function).
>
> **Discovery que o gateway monta** (confirme contra o token real): authority `https://<tenant>.ciamlogin.com/<Jwt__CiamTenantId>`, issuer `…/v2.0`. Validação fail-closed; `ClockSkew=0`.

**Save** → nova revisão.

### 4.5 Smoke test do gateway

```bash
GATEWAY=<Application Url do Container App>
curl -fsS $GATEWAY/health                         # → {"status":"healthy",...} (rota anônima)
curl -s -o /dev/null -w '%{http_code}' -X POST $GATEWAY/purchase \
  -H "Content-Type: application/json" -d '{"matchId":1,"category":"VIP","userId":1,"quantity":1}'
# → 401 (fail-closed: sem token o gateway recusa)
```

✅ **Checkpoint:** `/health` = 200; `POST /purchase` sem token = **401**. O gateway está no ar e fail-closed.

> 🧭 **As políticas do Bloco 1** (rate-limit 5/min → 429, output cache 30s em GET, CORS) já estão no código do gateway — demonstre-as agora que ele está no ar (repita um GET para ver o cache; estoure 5 POSTs/min para ver o 429).

---

## Fase 5 — Frontend: plugar a authority CIAM e publicar

1. Ligue o **SCM Basic Auth `On`** no Web App do frontend e capture o `AZURE_FRONTEND_PUBLISH_PROFILE` **depois** disso.
2. Configure as Variables do fork (build do frontend):

   | Variable | Valor | ⚠️ Confirme |
   |---|---|---|
   | `VITE_CIAM_AUTHORITY` | `https://<tenant>.ciamlogin.com/` | contém `ciamlogin.com`, **não** `microsoftonline.com` |
   | `VITE_CIAM_CLIENT_ID` | Client ID da App Reg SPA (Fase 3) | — |
   | `GATEWAY_V2_URL` | Application Url do gateway (Fase 4.3) | — |
   | `FRONTEND_APP_NAME`, `BACKEND_URL`, `FUNCTION_V2_URL` | (como nas Oitavas) | — |

   > ⚠️ **Erro nº1:** authority em `login.microsoftonline.com` → `AADSTS50011`. Como `ciamlogin.com` é authority "non-AAD", o MSAL exige `knownAuthorities: ['<tenant>.ciamlogin.com']` (o `authV2.ts` já contempla).

3. **Run workflow `acao=frontend`**.

✅ **Checkpoint:** frontend publicado com a authority CIAM embutida.

---

## Fase 6 — Login CIAM + smoke ponta-a-ponta (clímax do Bloco 2)

1. Abra o frontend → **Entrar (v2)** → redireciona para `<tenant>.ciamlogin.com`.
2. **Sign-up self-service:** "Continuar com Google" **ou** email + **OTP**.
3. Faça uma compra (`POST /purchase`) — o SPA envia `Authorization: Bearer <token-CIAM>`.
4. Confirme no SQL:
   ```sql
   SELECT TOP 5 id, user_id, entra_oid, status, created_at
   FROM dbo.purchases WHERE entra_oid IS NOT NULL ORDER BY id DESC;
   ```

> 🔎 **GATE — confirme em runtime:** cole o access token em [jwt.ms](https://jwt.ms) e verifique o **formato exato** do `iss` (termina em `…/v2.0`), o `aud` (= seu client ID), e o claim `oid`. É aqui que você trava o formato do issuer CIAM e o `knownAuthorities`. **Nunca** cole tokens de produção — em sala é trial descartável.

✅ **Checkpoint (AC-11):** login CIAM → gateway valida → `X-Entra-OID` propagado → `purchases.entra_oid` (origem CIAM) gravado ao lado de registros v1.

> ☕ **PONTO DE PAUSA NATURAL** — se dividir em dois encontros, encerre aqui (cliente CIAM real validado pelo gateway = uma aula completa).

---

## Fase 7 — Admin workforce + App Role `Admin` (Bloco 3)

> Segundo mundo: o admin entra pelo **workforce**. ⚠️ Troque para o tenant **workforce** (domínio `*.onmicrosoft.com`, **não** `ciamlogin.com`).

### 7.1 App Registration admin

1. No tenant workforce: **Entra ID → App registrations → `New registration`** → Name `student-<iniciais>-admin` → single-tenant → **Register**.
2. Anote **Client ID** e **Directory (tenant) ID** (este é o **workforce**, diferente do CIAM).

### 7.2 App Role `Admin`

1. Na App Reg admin → **App roles → `Create app role`**: Display `Admin` · Members **Users/Groups** · **Value `Admin`** · habilitada → **Apply**.
2. Atribua: **Enterprise applications →** `student-<iniciais>-admin` **→ Users and groups → `Add user/group`** → seu admin → role `Admin` → **Assign**.

### 7.3 Gateway aceita o 2º emissor

No Container App do gateway, adicione App Settings:

| App Setting | Valor |
|---|---|
| `Jwt__AdminTenantId` | Tenant ID do **workforce** (7.1) |
| `Jwt__AdminClientId` | Client ID da App Reg admin (7.1) |

> Authority do admin: `https://login.microsoftonline.com/<Jwt__AdminTenantId>/v2.0`. Mesma mecânica do CIAM (issuer-agnóstico). Save → nova revisão.

### 7.4 Login admin separado do cliente

1. Logue como admin (authority workforce). Em [jwt.ms](https://jwt.ms): `iss = login.microsoftonline.com/.../v2.0` e `roles: ["Admin"]`.
2. 🧪 Um **cliente CIAM válido** numa rota admin recebe **403** (autenticado, sem a role) — não 401.

✅ **Checkpoint (AC-13):** login admin via workforce com `roles:["Admin"]`, separado do cliente CIAM. Dois mundos coexistindo.

---

## Fase 8 — Migração `users` v1 → CIAM (hands-on — Bloco 4, o clímax)

> A coluna `users.entra_oid` já existe (Fase 1) **vazia** — você a **preenche** aqui, de forma aditiva.

### 8.1 Listar os alvos
```sql
SELECT id, name, email, entra_oid FROM dbo.users WHERE entra_oid IS NULL ORDER BY id;
```

### 8.2 Sign-up no CIAM com o MESMO email do v1
Para cada conta, faça sign-up self-service no CIAM (Fase 6) com **o email idêntico** ao de `users` (Google ou email+OTP).

> 💡 **A lição:** a senha **bcrypt NÃO vai** pro CIAM (o External ID não importa hash). O usuário cria credencial nova; o `users.password` bcrypt fica **intacto** no caminho v1.

### 8.3 Capturar o `oid` emitido pelo CIAM
- **Via app:** logue e veja o `oid` no token (jwt.ms / DevTools).
- **Via Portal:** Entra admin center (tenant CIAM) → **Users** → o usuário → **Object ID** (= o `oid`).

### 8.4 Vincular o `oid` ao registro v1 (idempotente)
```sql
UPDATE dbo.users
SET    entra_oid = @oid       -- oid do 8.3
WHERE  email = @email         -- MESMO email do v1
  AND  entra_oid IS NULL;     -- guard de idempotência
```

### 8.5 Provar a coexistência (o clímax)
```sql
SELECT u.id AS user_id_v1, u.email,
       CASE WHEN u.password LIKE '$2%' THEN 'bcrypt-presente' ELSE 'sem-bcrypt' END AS credencial_v1,
       u.entra_oid AS oid_ciam_v2,
       CASE WHEN u.password IS NOT NULL AND u.entra_oid IS NOT NULL
            THEN 'COEXISTE (v1 bcrypt + v2 CIAM)'
            WHEN u.entra_oid IS NULL THEN 'so v1 (nao migrou)'
            ELSE 'estado inesperado' END AS status_migracao
FROM dbo.users u WHERE u.email = @email;
```
Esperado: `status_migracao = COEXISTE (v1 bcrypt + v2 CIAM)`.

✅ **Checkpoint (AC-16):** uma linha de `users` com as **duas identidades** vivas lado a lado. Modernização sem destruição, provada em banco.

---

## Validação final (DoD)

- [ ] Conta Google do lab + Gemini key guardada (p/ último lab)
- [ ] Migrations aplicadas (`users.entra_oid` existe, vazia)
- [ ] Tenant CIAM (`ciamlogin.com`) + user flow + Google/OTP funcionando
- [ ] App Reg SPA no CIAM; `VITE_CIAM_AUTHORITY` = `<tenant>.ciamlogin.com`
- [ ] Gateway no ar: `/health` 200 + `POST /purchase` sem token = 401
- [ ] Login CIAM → `purchases.entra_oid` (origem CIAM) gravado ao lado do v1
- [ ] App Reg admin (workforce) + App Role `Admin`; cliente CIAM em rota admin = 403
- [ ] Migração executada; coexistência `COEXISTE (v1 bcrypt + v2 CIAM)` provada
- [ ] Nenhum recurso **Azure AD B2C** (`b2clogin.com`) criado

---

## Apêndice A — OAuth Client do Google (login social do CIAM)

> ⚠️ O Google reorganizou esta área: o antigo "OAuth consent screen" virou **Google Auth Platform** (Branding / Audience / Clients). Faça **depois** de ter o **Tenant ID** do CIAM (Fase 2.1) — os redirect URIs dependem dele.

### A.1 Projeto + tela de consentimento
1. [console.cloud.google.com](https://console.cloud.google.com) (conta do lab) → seletor de projeto → **New Project** (ex.: `fifa2026-ciam-lab`) → **Create** → selecione-o.
2. **☰ → Google Auth Platform → Branding** (`.../auth/branding`). Na 1ª vez abre o wizard **Get started**:
   - **App Information:** App name + User support email (o Gmail do lab).
   - **Audience:** **External** (conta Gmail comum só tem External).
   - **Contact Information:** email → conclua.
3. **Google Auth Platform → Audience:** confirme **Publishing status = Testing** e adicione o Gmail do lab em **Test users**.
4. Em **Branding → Authorized domains**, adicione **`ciamlogin.com`** e **`microsoftonline.com`**.

### A.2 Criar o OAuth client (Web application)
1. **☰ → Google Auth Platform → Clients** (`.../auth/clients`) → **Create client**.
2. **Application type:** **Web application** · **Name:** `entra-ciam-callback`.
3. **Authorized redirect URIs** → adicione **TODOS** estes (a doc da Microsoft lista ambos os formatos; substitua `<tenant-ID>` pelo Directory ID e `<tenant-subdomain>` pelo subdomínio):
   - `https://login.microsoftonline.com`
   - `https://login.microsoftonline.com/te/<tenant-ID>/oauth2/authresp`
   - `https://login.microsoftonline.com/te/<tenant-subdomain>.onmicrosoft.com/oauth2/authresp`
   - `https://<tenant-ID>.ciamlogin.com/<tenant-ID>/federation/oidc/accounts.google.com`
   - `https://<tenant-subdomain>.ciamlogin.com/<tenant-ID>/federation/oauth2`
   - `https://<tenant-subdomain>.ciamlogin.com/<tenant-subdomain>.onmicrosoft.com/federation/oauth2`
4. **Create** → **copie o Client ID e o Client secret** (⚠️ o secret só aparece **agora** por inteiro).
5. Leve Client ID + secret para a Fase 2.3 (federação no Entra).

> ⚠️ Mudanças nas credenciais podem levar de 5 min a horas para propagar. Divergência de redirect URI → `redirect_uri_mismatch`.

**Docs:** [Google federation no External ID](https://learn.microsoft.com/en-us/entra/external-id/customers/how-to-google-federation-customers) · [OAuth clients (Google)](https://support.google.com/cloud/answer/15549257)

---

## Apêndice B — Troubleshooting

| Sintoma | Causa provável | Mitigação |
|---|---|---|
| `AADSTS50011` no login cliente | authority com `microsoftonline.com` | `VITE_CIAM_AUTHORITY` = `<tenant>.ciamlogin.com` (Fase 5) |
| MSAL recusa authority "não confiável" | falta `knownAuthorities` | `knownAuthorities: ['<tenant>.ciamlogin.com']` |
| Gateway não sobe / Container App unhealthy | `Jwt__CiamTenantId` ausente (fail-closed) | configurar App Settings de identidade (Fase 4.4) |
| **502** em toda chamada | Target port ≠ 8080 | ingress = **8080** |
| **502** só em `/purchase` | `FunctionAppF1Url` ausente/errada | apontar p/ a Function F1 real |
| `redirect_uri_mismatch` (Google) | redirect URI do OAuth client ≠ callback do Entra | cadastrar **todos** os URIs do Apêndice A.3 |
| **401** no admin login | gateway sem `Jwt__AdminTenantId` | configurar 2º emissor (Fase 7.3) |
| `roles` ausente no token admin | role não atribuída | Enterprise applications → atribuir `Admin` (7.2) |
| Cliente CIAM em rota admin dá 401 (esperava 403) | policy fixando esquema | `AdminOnly` só `RequireRole("Admin")` (já no código) |
| Usuário não migra / `so v1` | UPDATE não rodou / email divergente | re-executar UPDATE idempotente (8.4) |
| Trial CIAM expirado (30d) | trial encerrou | recriar tenant External ID (ou converter p/ subscription, free 50K MAU) |

---

> **Em produção (e no CI/CD):** push, provisão de tenant/App Reg e smoke em Azure real são responsabilidade do **@devops**. Em sala, você faz à mão para entender cada peça.
</content>
