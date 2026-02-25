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
RUN printf '%%s\n' \
    "const express = require('express');" \
    "const path = require('path');" \
    "const app = express();" \
    "const PORT = process.env.PORT || 8080;" \
    "app.use(express.static(path.join(__dirname, 'dist')));" \
    "app.get('*', (req, res) => res.sendFile(path.join(__dirname, 'dist', 'index.html')));" \
    "app.listen(PORT, '0.0.0.0', () => console.log('Rodando na porta ' + PORT));" \
    > server.js
EXPOSE 8080
CMD ["node", "server.js"]