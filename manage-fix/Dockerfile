# Stage 1: Build
FROM node:22-slim AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y git ca-certificates --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# A MÁGICA ESTÁ AQUI: Puxando o repositório novo v2
RUN git clone --depth 1 https://github.com/EvolutionAPI/evolution-manager-v2.git .

RUN npm install
RUN npm run build

# Stage 2: Runtime
FROM node:22-slim
WORKDIR /app
RUN npm install express@4
COPY --from=builder /app/dist ./dist

# Criando o server.js blindado
RUN echo "const express = require('express');" > server.js && \
    echo "const path = require('path');" >> server.js && \
    echo "const app = express();" >> server.js && \
    echo "const PORT = process.env.PORT || 8080;" >> server.js && \
    echo "app.use(express.static(path.join(__dirname, 'dist')));" >> server.js && \
    echo "app.get('*', (req, res) => res.sendFile(path.join(__dirname, 'dist', 'index.html')));" >> server.js && \
    echo "app.listen(PORT, '0.0.0.0', () => console.log('Rodando na porta ' + PORT));" >> server.js

EXPOSE 8080
CMD ["node", "server.js"]