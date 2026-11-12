
FROM node:22-bookworm-slim

WORKDIR /app

ENV NODE_ENV=production
ENV NODE_OPTIONS=--max-old-space-size=160
# First Build Failure Fix, Copy package.json and package-lock.json separately to leverage Docker cache for dependencies

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY --chown=node:node . .

EXPOSE 8080

USER node

ENTRYPOINT ["sh", "./entrypoint.sh"]