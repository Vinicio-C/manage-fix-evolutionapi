# Usa a versão exata do Node que precisamos
FROM node:22-alpine

# Cria a pasta de trabalho
WORKDIR /app

# Instala o painel e a dependência fujona juntos
RUN npm install evolution-manager@0.4.13 fs-extra

# Avisa o Railway qual porta vamos usar
EXPOSE 9615

# Variáveis de ambiente
ENV NODE_ENV=production
ENV PORT=9615

# Comando correto para iniciar o servidor
CMD ["npx", "evolution-manager", "server", "start"]