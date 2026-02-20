# Usa a versão exata do Node que precisamos
FROM node:22-alpine

# Cria a pasta de trabalho
WORKDIR /app

# Instala o painel e a dependência fujona juntos
RUN npm install evolution-manager@0.4.13 fs-extra

# Avisa o Railway qual porta vamos usar
EXPOSE 8080

# Força as variáveis de ambiente corretas
ENV HOST=0.0.0.0
ENV PORT=8080
ENV NODE_ENV=production

# Comando para rodar a aplicação
CMD ["npx", "evolution-manager"]