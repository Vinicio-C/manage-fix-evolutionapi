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

# Mesmo symlink do builder: serve é um script Node que usa /usr/bin/env node
RUN ln -s /usr/local/bin/node /usr/bin/node

RUN npm install -g serve

COPY --from=builder /app/dist ./dist

EXPOSE 3000

# tcp://0.0.0.0 força binding em todas as interfaces (não só localhost)
# Railway injeta $PORT dinamicamente
CMD ["sh", "-c", "serve -s dist --listen tcp://0.0.0.0:${PORT:-3000}"]