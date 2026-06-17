# VPS

# Конфигурации

- ssh-keygen -t rsa -b 4096

генерирует пару ключей - приватный и публичный. Вот публичный мы закидываем в сервисы, а приватный хранится у нас. Это для аутентификации по ssh


# Подключение

ssh user1@95.174.92.149
ssh-keygen -R 95.174.92.149
type C:\Users\asv\.ssh\id_rsa.pub


# Dockerfile Next

```
FROM node:22-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

RUN mkdir .next && chown nextjs:nodejs .next

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT=3000

CMD ["node", "server.js"]

```
