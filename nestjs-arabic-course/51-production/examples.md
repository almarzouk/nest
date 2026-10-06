# أمثلة عملية

أربعة مسارات على نفس التطبيق: بناء يُشغَّل، بيئة تُحقَّق، سجل بلا سر، ووكيل ينهي TLS. الاستيراد في TypeScript ينتهي بـ `.js`.

## بناء ثم تشغيل

```bash
npm ci
npx prisma generate
npm run build
node dist/main.js
```

`start:dev` يبقى على جهازك. الصورة والأمر في المنصة يشيران إلى `dist/main.js`. لا تُودِع مجلد `dist` في Git.

## إقلاع يقرأ البيئة ويرفض الناقص

```ts
import { NestFactory } from '@nestjs/core';
import { Logger } from '@nestjs/common';
import helmet from 'helmet';
import { AppModule } from './app.module.js';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useLogger(new Logger());
  app.use(helmet());
  app.enableCors({ origin: ['https://shop.example'] });
  app.enableShutdownHooks();
  app.getHttpAdapter().getInstance().set('trust proxy', 1);
  await app.listen(process.env.PORT ?? 3000, '0.0.0.0');
}

void bootstrap();
```

`ConfigModule.forRoot` في الجذر يتحقق من `DATABASE_URL` قبل أن يصل الطلب. في الإنتاج `ignoreEnvFile: true` حتى لا يُقرأ `.env` نُسخ بالخطأ داخل الصورة. `.env.example` بلا كلمات سر.

## سجل عند الفشل

```ts
this.logger.error(`database unreachable requestId=${requestId}`);
```

لا تطبع `DATABASE_URL` ولا جسم تسجيل الدخول. المكدس في السجل على الخادم. العميل يرى رسالة عامة ورمز الحالة.

## Nginx أمام المنفذ الداخلي

```nginx
server {
  listen 443 ssl;
  server_name shop.example;
  client_max_body_size 10m;

  location / {
    proxy_pass http://api:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location /health/ {
    proxy_pass http://api:3000;
    proxy_set_header Host $host;
    add_header Cache-Control "no-store";
  }
}
```

الضغط في `gzip` هنا أو عبر `compression` على محول Express، لا الاثنين. اسم `api` هو خدمة Compose لا `localhost`.

## ماذا تتأكد منه قبل أول نشر

1. الأمر `node dist/main.js` والمستخدم ليس root داخل الصورة.
2. `enableShutdownHooks()` و `onModuleDestroy` يقطع Prisma و Redis.
3. أصل CORS واحد تعرفه، و `ValidationPipe` مع `whitelist`.
4. `trust proxy` يساوي عدد الوكلاء الفعلي، هنا `1`.
5. نسختان أو أكثر لا تشاركان جلسة في مصفوفة داخل العملية.
6. مسار الصحة بلا كاش في الوكيل، والفصل التالي يفصل الحياة عن الجاهزية.
