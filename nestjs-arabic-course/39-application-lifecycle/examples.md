# أمثلة عملية

## مثال 1 — Prisma يفتح في الأوراق ويغلق في الإيقاف

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

`PrismaModule` يصدّر الخدمة وتستورده الميزات. لأنه أعمق من الجذر، `$connect` يُنتظر قبل أن تكمل الوحدات الأعلى `onModuleInit`. في الإيقاف تمشي الوحدات الأعلى أولًا، ثم تصل الورقة إلى `$disconnect`.

## مثال 2 — بوابة دفع تتصل بعد اكتمال الرسم

```ts
import { Inject, Injectable, OnApplicationBootstrap, OnModuleDestroy } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';

@Injectable()
export class PaymentsBus implements OnApplicationBootstrap, OnModuleDestroy {
  constructor(@Inject('PAYMENTS') private readonly client: ClientProxy) {}

  async onApplicationBootstrap(): Promise<void> {
    await this.client.connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.client.close();
  }
}
```

`connect` يحجز `listen` حتى يقوم الناقل. `close` في أول خطاف إيقاف يحرر العميل قبل خطوات قطع HTTP. إن كان الدفع اختياريًا، لا تترك الرفض يوقف المتجر كله.

## مثال 3 — علم إفراغ ثم سجل بعد قطع المقابس

```ts
import {
  BeforeApplicationShutdown,
  Injectable,
  Logger,
  OnApplicationShutdown,
} from '@nestjs/common';

@Injectable()
export class ShutdownTrace
  implements BeforeApplicationShutdown, OnApplicationShutdown
{
  private readonly logger = new Logger(ShutdownTrace.name);

  beforeApplicationShutdown(signal?: string): void {
    this.logger.log(`drain ${signal ?? 'close'}`);
  }

  onApplicationShutdown(signal?: string): void {
    this.logger.log(`sockets closed ${signal ?? 'close'}`);
  }
}
```

السطر الأول قبل إغلاق الاتصالات، والثاني بعده. كلاهما يحتاج مسار إنهاء حقيقي.

## مثال 4 — الإقلاع الذي يسمع الإشارة

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks();
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

على Linux، `SIGTERM` يدخل السلسلة. اختبار ينادي `app.close()` يمر بالخطافات حتى لو حذفت سطر التفعيل، لأن النداء الصريح مسار مستقل.

## مثال 5 — مؤقت يُمسح مع الإيقاف

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';

@Injectable()
export class BillingClock implements OnModuleInit, OnModuleDestroy {
  private timer?: NodeJS.Timeout;

  onModuleInit(): void {
    this.timer = setInterval(() => this.tick(), 60_000);
  }

  onModuleDestroy(): void {
    if (this.timer) clearInterval(this.timer);
  }

  private tick(): void {
    return;
  }
}
```

`onModuleInit` هنا متزامن. الإطار ينتظره أيضًا. `clearInterval` يمنع بقاء العملية بعد `app.close()`.

## ماذا تتأكد منه

1. فشل `$connect` يمنع فتح المنفذ.
2. Ctrl+C يطبع `drain` ثم `sockets closed` بعد التفعيل.
3. خدمة `Scope.REQUEST` لا تطبع أي خطاف تطبيق.
4. `setInterval` من غير `clearInterval` يبقي العملية بعد `close`.
