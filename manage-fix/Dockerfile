# node:22-slim (Debian) evita o problema de PATH do Alpine com npm em subprocessos
FROM node:22-slim

WORKDIR /app

# Instala o painel e a dependência auxiliar
RUN npm install evolution-manager@0.4.13 fs-extra

# Instala as devDependencies do evolution-manager (necessário para o build do Vite no startup)
RUN cd node_modules/evolution-manager && npm install --include=dev

EXPOSE 9615

ENV NODE_ENV=production
ENV PORT=9615

CMD ["npx", "evolution-manager", "server", "start"]