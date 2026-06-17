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

# docker compose next .net pg

```
networks:
  home-network:
    driver: bridge

volumes:
    postgres_data:

services:
  next:
    container_name: ContainerNext
    build:
      context: ./school.next
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
    ports:
      - 80:3000 
    networks:
      - home-network   
  api:
    container_name: ContainerAPI
    build:
      context: .
      dockerfile: PodlesnoeSchoolApp.API/Dockerfile
    ports:
      - 5001:5000
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://0.0.0.0:5000
      - ConnectionStrings__DefaultConnection=Host=db;Port=5432;Database=PodlesnoeAppDatabase;Username=postgres;Password=root 
    depends_on:
      - db
    networks:
      - home-network
  db:
    container_name: ContainerPostgreSQL
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: root
      POSTGRES_DB: PodlesnoeAppDatabase
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - home-network    
   
```


# Очистка

docker system prune -a --volumes

# Права всем пользователям для папки

Замечание: На локальной машине папка ./uploads была создана от вашего пользователя (UID 1000), который совпадает с пользователем app в контейнере. На VPS вы создали папку через sudo (от root), поэтому владелец — root, а не app. Контейнер не может писать в папку, которой владеет root, даже если права 777 (потому что это мера безопасности Docker).

`sudo chmod -R 777 ./uploads`

всегда создавайте папки для монтирования от текущего пользователя, а не от root
