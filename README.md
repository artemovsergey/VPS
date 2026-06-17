# VPS

# Конфигурации

- ssh-keygen -t rsa -b 4096

генерирует пару ключей - приватный и публичный. Вот публичный мы закидываем в сервисы, а приватный хранится у нас. Это для аутентификации по ssh


# Подключение

ssh user1@95.174.92.149
ssh-keygen -R 95.174.92.149
type C:\Users\asv\.ssh\id_rsa.pub

# nginx.conf

```
events {
    worker_connections 1024;
}

http {
    upstream react {
        server react:80;
    }
    
    upstream api {
        server api:5000;
    }
    
    client_max_body_size 100M;

    server {
        listen 80;
        server_name localhost;
        
        location /api/ {
            proxy_pass http://api;
            # proxy_set_header Host $host;
            # proxy_set_header X-Real-IP $remote_addr;
            # proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            # proxy_set_header X-Forwarded-Proto $scheme;
        }

        location / {
            proxy_pass http://react;
            # proxy_set_header Host $host;
            # proxy_set_header X-Real-IP $remote_addr;
            # proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            # proxy_set_header X-Forwarded-Proto $scheme;
            
            # # Важно для SPA - перенаправляем все запросы на index.html
            # proxy_set_header X-Original-URI $request_uri;
        }
        

    }
}
```

# dockerfile nginx

```
FROM nginx:alpine
COPY ./loadbalancer/nginx.conf /etc/nginx/nginx.conf
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]

```

# dockerfile react nginx

```
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine
# Копируем сборку
COPY --from=build /app/dist /usr/share/nginx/html
# Создаем конфиг Nginx
RUN echo 'server { \
    listen 80; \
    root /usr/share/nginx/html; \
    index index.html; \
    location / { \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```


# dockerfile next

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


# docker compose react .net pg nginx
```
networks:
  home-network:
    driver: bridge

volumes:
    postgres_data:
    # uploads_data:

services:
  nginx:
    container_name: ContainerNginx
    build: 
      context: .
      dockerfile: loadbalancer/Dockerfile
    restart: always
    ports:
      - "80:80"
      - "443:443"
    links:
      - api
      - react
    depends_on:
      - db
      - api
      - react
    networks:
      - home-network 
  react:
    container_name: ContainerReact
    build:
      context: ./lokovapp.react
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
    ports:
      - 3001:80 
    networks:
      - home-network   
  api:
    container_name: ContainerAPI
    image: lokovapp
    build:
      context: .
      dockerfile: LokovApp/Dockerfile
    ports:
      - 5001:5000
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://0.0.0.0:5000
      - ConnectionStrings__DefaultConnection=Host=db;Port=5432;Database=LokovAppDatabase;Username=postgres;Password=root
    volumes:
      - ./uploads:/app/uploads   # ← bind mount (точка слеш)
    depends_on:
      - db
    networks:
      - home-network
  db:
    container_name: ContainerPostgres
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: root
      POSTGRES_DB: LokovAppDatabase
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - home-network    
   
```
