# أمثلة عملية

ثلاثة تراكيب على مكتب إعارة. كل مثال يضيف حدًا واحدًا من هذا الفصل: دالة، ثم صنف بحقن، ثم استثناء صحة. الاستيراد المحلي ينتهي بـ `.js`.

## مثال 1 — معرّف على كل طلب

`request-id.middleware.ts` دالة بلا اعتماديات. `main.ts` يمررها إلى المنصة فتعمل قبل أي `configure`.

```ts
import { randomUUID } from 'node:crypto';
import type { NextFunction, Request, Response } from 'express';

export function requestId(req: Request, res: Response, next: NextFunction): void {
  const header = req.header('x-request-id');
  const id = header && header.length > 0 ? header : randomUUID();
  res.setHeader('x-request-id', id);
  next();
}
```

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';
import { requestId } from './request-id.middleware.js';

const app = await NestFactory.create(AppModule);
app.use(requestId);
await app.listen(3000);
```

طلب بلا الترويسة يخرج بـ UUID. طلب البوابة يحتفظ بقيمته. لا `providers` هنا.

## مثال 2 — ساعة تُحقن على مسارات النسخ

`CopyClock` مزوّد. `CopyClockMiddleware` يلصق الوقت. الوحدة تطبّق `NestModule` ولا تضع الصنف في `providers`.

```ts
import { Injectable, type NestMiddleware } from '@nestjs/common';
import type { NextFunction, Request, Response } from 'express';
import { CopyClock } from './copy-clock.js';

@Injectable()
export class CopyClockMiddleware implements NestMiddleware {
  constructor(private readonly clock: CopyClock) {}

  use(_req: Request, res: Response, next: NextFunction): void {
    res.setHeader('x-copy-time', this.clock.iso());
    next();
  }
}
```

```ts
import { Module, type MiddlewareConsumer, type NestModule } from '@nestjs/common';
import { CopiesController } from './copies.controller.js';
import { CopyClock } from './copy-clock.js';
import { CopyClockMiddleware } from './copy-clock.middleware.js';

@Module({
  controllers: [CopiesController],
  providers: [CopyClock],
})
export class CopiesModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer.apply(CopyClockMiddleware).forRoutes(CopiesController);
  }
}
```

اختبار الوحدة يستبدل `CopyClock`. مسارات الأعضاء لا ترى الترويسة لأن المتحكم غير مذكور في `forRoutes`.

## مثال 3 — تتبع المكتب مع ثقب الصحة

`DeskModule` تغطي مساراتها وتطرح `GET /health` و `GET /metrics`.

```ts
import {
  Module,
  RequestMethod,
  type MiddlewareConsumer,
  type NestModule,
} from '@nestjs/common';
import { DeskController } from './desk.controller.js';
import { traceDesk } from './trace-desk.middleware.js';

@Module({ controllers: [DeskController] })
export class DeskModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer
      .apply(traceDesk)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
        { path: 'metrics', method: RequestMethod.GET },
      )
      .forRoutes(DeskController);
  }
}
```

`traceDesk` دالة تشبه `requestId` لكنها تكتب `x-desk: circulation` بعد أن ضمن المثال 1 وجود المعرّف. لأنها في وحدة مستوردة من الجذر، تعمل بعد `app.use`.

على `GET /health` داخل هذا المتحكم لا تُكتب `x-desk`. على `GET /loans` تُكتب. `app.use(requestId)` ما زال يعمل على الصحة لأن الاستثناء يخص سلسلة `DeskModule` فقط.

## ترتيب طلب واحد

`GET /copies/9` يمر على `requestId` في `app.use`، ثم على `CopyClockMiddleware` لأن `CopiesController` مذكور. لا يمر على `traceDesk` إن كان النسخ خارج `DeskController`.

`POST /loans` داخل المكتب يمر على `requestId` ثم على `traceDesk` إن لم يُستثنَ الفعل. `GET /health` يمر على `requestId` فقط من هذه الأمثلة، لأن `exclude` طرحه من سلسلة المكتب.

## ماذا تتأكد منه

- كل `next()` حاضر، أو الرد منتهٍ في الدالة نفسها.
- الدالة في `apply` أو `app.use`، والصنف ذو الاعتماد في `apply(Class)` مع مزوّده مرئيًا في الوحدة.
- `exclude` يسبق `forRoutes` ويطرح من تلك السلسلة فقط.
- لا فحص أدوار داخل هذه الملفات. الحارس فصل لاحق.
- خطأ يُرمى من `configure`/`apply` لا يُمسك بفلتر على المتحكم.
