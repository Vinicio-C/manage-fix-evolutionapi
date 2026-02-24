# node:22-slim (Debian) — sem os problemas de PATH do Alpine
FROM node:22-slim

WORKDIR /app

# Vite usa #!/usr/bin/env node mas no slim o node está só em /usr/local/bin
# Este symlink resolve o "No such file or directory" no subprocesso de build
RUN ln -s /usr/local/bin/node /usr/bin/node

# Vite global: utilizado pelo evolution-manager no build step ao iniciar
RUN npm install -g vite

# Instala o painel
RUN npm install evolution-manager@0.4.13 fs-extra

EXPOSE 9615

ENV NODE_ENV=production
ENV PORT=9615

CMD ["npx", "evolution-manager", "server", "start"]