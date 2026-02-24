# ── Stage 1: Build ───────────────────────────────────────────────────────────
# Constrói o app Vue durante o docker build (evita rebuild problemático no startup)
FROM node:22-slim AS builder
WORKDIR /app

# node está em /usr/local/bin mas scripts usam /usr/bin/env node
RUN ln -s /usr/local/bin/node /usr/bin/node

# Vite global para executar o build
RUN npm install -g vite

# Fixa vuetify@3.5.0: vite-plugin-vuetify@1.0.2 (usado pelo evolution-manager@0.4.13)
# só é compatível com vuetify <3.8.0 — versões >=3.8 mudaram os caminhos de importação
RUN npm install evolution-manager@0.4.13 vuetify@3.5.0 vite-plugin-vuetify@1.0.2 fs-extra --legacy-peer-deps

# Executa o build do Vue dentro do pacote
RUN cd node_modules/evolution-manager && npx vite build

# ── Stage 2: Runtime ──────────────────────────────────────────────────────────
# Apenas serve os arquivos estáticos já construídos
FROM node:22-slim
WORKDIR /app

RUN npm install -g serve

# Copia só o dist construído — sem node_modules volumosos
COPY --from=builder /app/node_modules/evolution-manager/dist ./dist

EXPOSE 9615

ENV PORT=9615

CMD ["serve", "-s", "dist", "-l", "9615"]