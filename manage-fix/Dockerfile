# ── Stage 1: Build ───────────────────────────────────────────────────────────
# Clona o repositório git com a tag exata → traz o package-lock.json original
# npm ci instala as versões EXATAS testadas pelo autor, sem drift de dependências
FROM node:22-slim AS builder
WORKDIR /app

RUN apt-get update && apt-get install -y git ca-certificates --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 --branch 0.4.13 \
    https://github.com/gabrielpastori1/evolution-manager.git .

# npm ci usa o package-lock.json exato do repo
RUN npm ci --legacy-peer-deps

RUN npm run build

# ── Stage 2: Runtime ──────────────────────────────────────────────────────────
FROM node:22-slim
WORKDIR /app

RUN npm install -g serve

COPY --from=builder /app/dist ./dist

EXPOSE 9615

ENV PORT=9615

CMD ["serve", "-s", "dist", "-l", "9615"]