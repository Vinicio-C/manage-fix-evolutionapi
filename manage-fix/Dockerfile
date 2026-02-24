# ── Stage 1: Build ───────────────────────────────────────────────────────────
# Clona o repositório git com a tag exata → traz o package-lock.json original
# npm ci instala as versões EXATAS testadas pelo autor, sem drift de dependências
FROM node:22-slim AS builder
WORKDIR /app

RUN apt-get update && apt-get install -y git ca-certificates --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 \
    https://github.com/EvolutionAPI/evolution-manager.git .

# npm ci usa o package-lock.json exato do repo
RUN npm ci --legacy-peer-deps

RUN npm run build

# ── Stage 2: Runtime ──────────────────────────────────────────────────────────
FROM node:22-slim
WORKDIR /app

# Express para servir os arquivos estáticos — sem CLI intermediária
RUN npm install express@4

COPY --from=builder /app/dist ./dist

# Gera server.js inline: evita dependência do build context do Railway
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