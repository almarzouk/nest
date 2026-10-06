# ماذا بعد TypeScript؟

تستطيع الآن قراءة السطر الذي يتكرر في الإطار، وتعرف أن النوع وحده لا يفحص JSON، وأن الزينة تخزن وصفًا، وأن `reflect-metadata` مع `emitDecoratorMetadata` يبقيان نوع الصنف للمعامل.

الخطوة التالية تشغيل هذا على مشروع حقيقي. قالب Nest يولد `tsconfig` وفيه `strict` والزينة، وملف `main.ts` فيه:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

الفصل التالي يشرح متطلبات Node، و `nest new`، و ESM، ووظيفة كل ملف في `src/`، من غير الغوص بعد في معنى كل خاصية داخل `@Module`.

[التثبيت وهيكل المشروع](../02-project-setup/README.md)
