# أمثلة عملية

## إقلاع القالب

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

1. تحميل الملف يلصق Metadata.
2. `create` يبني الجذر `AppModule` والحاوية ويربط `GET /` عبر محوّل Express.
3. `listen` يفتح 3000 إن لم يوجد `PORT`.
4. الطلب يستخدم نسخة `AppService` التي بُنيت مع السياق لا نسخة جديدة.

## فشل متعمد تتعلم منه

احذف `AppService` من `providers` مؤقتًا وشغّل `start:dev`. الإقلاع يتوقف لأن الحاوية لا تحل اعتماد المتحكم. أرجع السطر. هذا أثر `abortOnError` الافتراضي: العملية لا تبقى متاحة بنصف حاوية.

## رسم صغير

```text
AppController -> AppService
```

ورقة واحدة، ولا عبور وحدات بعد.

## ماذا بعد؟

تمارين الفصل ثم Modules.
