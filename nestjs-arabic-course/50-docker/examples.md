# أمثلة عملية

## تشغيل إنتاجي داخل الصورة

```dockerfile
FROM node:22.14-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22.14-bookworm-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

## شبكة الأسماء

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://app:app@postgres:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      retries: 10
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7
volumes:
  pgdata:
```

## الإقلاع الذي يرى الإشارة

```ts
app.enableShutdownHooks();
await app.listen(process.env.PORT ?? 3000, '0.0.0.0');
```

`$disconnect` و `quit` في `onModuleDestroy`.

## ماذا تتأكد منه

1. `DATABASE_URL` من داخل الحاوية يحتوي `postgres` لا `localhost`.
2. `docker image` لا تُبنى بـ `start:dev`.
3. `compose down` لا يحذف `pgdata` إلا مع `-v`.
4. وسم Node يحقق 22.12+ أو 20.19+.
