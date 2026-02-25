# Contexto do Projeto — WhatsApp CRM no Railway

> Documento gerado em 2026-02-25 para continuidade da sessão no Gemini.

---

## Visão Geral

Projeto de **WhatsApp CRM** deployado na plataforma **Railway**, composto pelos serviços:

| Serviço | Função |
|---|---|
| **Evolution API v2.3.7** | Backend WhatsApp (protocolo Baileys) |
| **Evolution Manager** | UI web para gerenciar instâncias da Evolution API |
| **n8n** | Orquestrador de fluxos / automação |
| **Typebot** | Chatbot visual (builder + viewer) |
| **PostgreSQL** | Banco de dados compartilhado |
| **Redis** | Cache / persistência de sessões |
| **Gemini AI** | IA usada dentro dos fluxos n8n |
| **Telegram** | Canal alternativo de notificações |

---

## Repositório Local

```
c:\Users\USUARIO\Desktop\DESENVOLVIMENTO\PROJETOS\evolution-api\
├── correcoes evolution.md   ← histórico detalhado das correções
├── manage-fix/
│   ├── Dockerfile           ← build customizado do Evolution Manager
│   ├── server.js            ← servidor Express para SPA estático
│   └── package.json
└── CONTEXTO-PARA-GEMINI.md  ← este arquivo
```

---

## Problema Principal: Evolution Manager no Railway

O **Evolution Manager** é uma SPA (Vue/Vite). O Railway tentava executá-lo com `npm start` via `package.json` do próprio projeto, o que falhava porque o build não existia em runtime.

### Solução — Dockerfile customizado (`manage-fix/Dockerfile`)

**Estratégia:** multi-stage build

1. **Stage 1 (builder):** clona o repositório oficial do Evolution Manager via `git clone`, instala dependências com `npm ci --legacy-peer-deps`, roda `npm run build` → gera `/app/dist`
2. **Stage 2 (runtime):** node:22-slim, instala apenas `express@4`, copia `/app/dist`, gera `server.js` inline que serve a SPA estática com fallback para `index.html`

```dockerfile
# Stage 1: Build
FROM node:22-slim AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y git ca-certificates --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*
RUN git clone --depth 1 https://github.com/EvolutionAPI/evolution-manager.git .
RUN npm ci --legacy-peer-deps
RUN npm run build

# Stage 2: Runtime
FROM node:22-slim
WORKDIR /app
RUN npm install express@4
COPY --from=builder /app/dist ./dist
RUN printf '%s\n' \
    "const express = require('express');" \
    "const path = require('path');" \
    "const app = express();" \
    "const PORT = process.env.PORT || 3000;" \
    "app.use(express.static(path.join(__dirname, 'dist')));" \
    "app.get('*', (req, res) => res.sendFile(path.join(__dirname, 'dist', 'index.html')));" \
    "app.listen(PORT, '0.0.0.0', () => console.log('Rodando na porta ' + PORT));" \
    > server.js
EXPOSE 3000
CMD ["node", "server.js"]
```

**Por que `RUN printf` em vez de `COPY server.js`?**
O Railway faz build a partir do GitHub. Colocar o `server.js` no `RUN` evita depender de que o arquivo exista no contexto de build — tudo é gerado na própria imagem.

**server.js (também versionado separadamente):**
```js
const express = require('express');
const path = require('path');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.static(path.join(__dirname, 'dist')));
app.get('*', (req, res) => res.sendFile(path.join(__dirname, 'dist', 'index.html')));
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Evolution Manager rodando em http://0.0.0.0:${PORT}`);
});
```

---

## Problemas Resolvidos — Todos os Serviços

### 1. Typebot 502 "Application failed to respond"

**Sintoma:** Deploy "Online", logs mostravam `Ready in 904ms`, mas HTTP 502 em TODOS endpoints (incluindo arquivos estáticos).

**Root cause:** Next.js 15 mudou o bind padrão de `0.0.0.0` para o IP do container. O Railway proxy não conseguia alcançar o app.

**Fix:** Adicionar variável de ambiente no serviço `typebot-builder`:
```
HOSTNAME=0.0.0.0
```

**Resultado:** HTTP 200 — Typebot online.
**Mesmo fix aplicado ao** `typebot-viewer`.

---

### 2. Typebot — "Configure pelo menos um provedor de autenticação"

**Sintoma:** Login no Typebot mostrava erro de auth provider.

**Causa:** Sem SMTP configurado, Typebot requer OAuth.

**Fix:** Criar GitHub OAuth App e configurar:
```
GITHUB_CLIENT_ID=<id>
GITHUB_CLIENT_SECRET=<secret>
```
Callback URL obrigatória: `{NEXTAUTH_URL}/api/auth/callback/github`

---

### 3. Evolution API — Crash P3005 (Prisma)

**Sintoma:** `prisma migrate deploy` falhava com P3005 — "Database schema is not empty".

**Causa:** PostgreSQL compartilhado já tinha tabelas no schema `public` (do Typebot).

**Fix:** Adicionar `?schema=evolution_api` na connection string:
```
DATABASE_CONNECTION_URI=postgresql://user:pass@host/db?schema=evolution_api
```

**Nota técnica:** A Evolution API usa `gen_random_uuid()` (nativo do Postgres), não `uuid_generate_v4()`, então schema customizado funciona sem extensões adicionais.

---

### 4. Evolution API — Instâncias desaparecendo

**Sintoma:** Instância criada sumia após ~30 segundos.

**Causa real:** O crash P3005 reiniciava o container. Após fix do schema, instâncias passaram a persistir.

---

### 5. Evolution API v2.3.7 — Mudanças de API

A versão 2.3.7 quebrou scripts que assumiam formato antigo:

| Endpoint | Formato antigo | Formato v2.3.7 |
|---|---|---|
| `fetchInstances` | Retorna array direto `[...]` | Retorna `{value: [...]}` |
| `setWebhook` | `{url, enabled, events}` | `{webhook: {url, enabled, events}}` |
| Evento webhook | `messages.upsert` (lowercase dot) | `MESSAGES_UPSERT` (uppercase underscore) |

**Fix no script de conexão:**
```powershell
function Get-Instances {
    $r = Invoke-RestMethod -Uri "$EVO_URL/instance/fetchInstances" -Headers $HDR
    if ($r.value) { return $r.value }  # v2.3.7
    if ($r -is [array]) { return $r }  # versões antigas
    return @()
}
```

**Fix no corpo do webhook:**
```powershell
$body = @{
    webhook = @{
        enabled         = $true
        url             = $N8N_WHK
        webhookByEvents = $false
        webhookBase64   = $false
        events          = @('MESSAGES_UPSERT', 'MESSAGES_UPDATE', 'CONNECTION_UPDATE')
    }
} | ConvertTo-Json -Depth 5
```

---

### 6. n8n — Filtro bloqueando todas as mensagens

**Sintoma:** n8n recebia webhook mas parava no nó "Ignorar: proprias msgs e eventos".

**Causa:** Filtro comparava `$json.event == "messages.upsert"` mas a Evolution API v2.3.7 envia `MESSAGES_UPSERT`.

**Fix no JSON do workflow n8n:**
```json
"leftValue": "={{ $json.event?.toLowerCase()?.replace('_', '.') }}",
"rightValue": "messages.upsert",
"operator": { "type": "string", "operation": "equals" }
```

---

## Fluxo WhatsApp → n8n → Typebot

```
Telefone → WhatsApp
              ↓
        Evolution API (instância "Evoluido")
              ↓ webhook MESSAGES_UPSERT
             n8n
              ↓ filtro: não é própria msg + é messages.upsert
        [Lógica de roteamento]
              ↓
           Typebot (slug: my-typebot-w42986e)
              ↓
        Gemini AI (dentro do fluxo Typebot/n8n)
```

---

## Configurações de Ambiente Críticas

### Evolution API
```env
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://...?schema=evolution_api
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://...
DATABASE_SAVE_DATA_INSTANCE=true
DATABASE_SAVE_DATA_NEW_MESSAGE=true
DATABASE_SAVE_MESSAGE_UPDATE=true
DATABASE_SAVE_DATA_CONTACTS=true
DATABASE_SAVE_DATA_CHATS=true
```

### Typebot Builder
```env
HOSTNAME=0.0.0.0
NEXTAUTH_URL=https://<sua-url-typebot>
NEXTAUTH_SECRET=<32-chars>
ENCRYPTION_SECRET=<32-chars>
DATABASE_URL=postgresql://...
GITHUB_CLIENT_ID=<id>
GITHUB_CLIENT_SECRET=<secret>
NODE_ENV=production
PORT=3000
```

### n8n
```env
NODE_FUNCTION_ALLOW_BUILTIN=*
```

---

## Estado Atual

| Item | Status |
|---|---|
| Typebot Builder (HTTP) | ✅ Online |
| Typebot Viewer (HTTP) | ✅ Online |
| Typebot Auth (GitHub OAuth) | ✅ Funcionando |
| Evolution API (P3005) | ✅ Resolvido |
| Evolution API instância "Evoluido" | ✅ Persistindo e conectada |
| Evolution API webhook → n8n | ✅ Funcionando |
| WhatsApp conectado (QR scaneado) | ✅ Status `open` |
| n8n filtro de eventos | ⚠️ Fix aplicado ao JSON — confirmar se foi salvo/reimportado corretamente no n8n |
| Evolution Manager (SPA no Railway) | ✅ Dockerfile customizado pronto, deployado |

---

## Próximos Passos (pendentes)

1. **Verificar n8n filter:** Abrir o workflow no n8n, confirmar que a expressão do filtro de evento usa `toLowerCase().replace('_', '.')`. Salvar explicitamente antes de testar.
2. **Testar fluxo completo:** Enviar mensagem de outro telefone → verificar que n8n processa → Typebot responde.
3. **Configurar Gemini AI** dentro do fluxo n8n/Typebot conforme necessidade do CRM.

---

## Observações Técnicas Gerais

- **Railway private networking:** Use `*.railway.internal` para comunicação interna entre serviços (sem expor porta pública).
- **PowerShell no Railway:** Scripts PS1 locais usados para gerenciar variáveis via Railway API GraphQL. Evitar caracteres Unicode especiais (em dash `—`, aspas curvas) — causam erros de parse.
- **Prisma + PostgreSQL compartilhado:** Sempre isolar com `?schema=nome_servico` para evitar P3005.
- **Next.js 15:** Sempre definir `HOSTNAME=0.0.0.0` em containers.
