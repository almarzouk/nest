# أمثلة عملية

## دورة ملفات تختفي بتغيير الاستيراد

`books/index.ts` يصدّر `BooksService` و `BooksController`. المتحكم كان يستورد الخدمة من `./index.js`. غيّره إلى:

```ts
import { BooksService } from './books.service.js';
```

لا دورة منطق هنا. البرميل كان يعيد التحميل. بعد الإصلاح احذف أي `forwardRef` أضفته قبل أن تتأكد من الرسالة.

## دورة خدمتين تُفك بوسيط

قبلًا `BillingService` تحقن `OrdersService.findTotal`. بعدها:

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class BillingService {
  open(orderId: string, totalMinor: number): { orderId: string; totalMinor: number } {
    return { orderId, totalMinor };
  }
}
```

`OrdersService` تستدعي `open` وتمرر المجموع. `BillingModule` لا يستورد `OrdersModule`.

## غلاف مؤقت على الطرفين

إن بقي حقن متبادل في هذا السحب:

```ts
import { Inject, Injectable, forwardRef } from '@nestjs/common';
import { BillingService } from '../billing/billing.service.js';

@Injectable()
export class OrdersService {
  constructor(
    @Inject(forwardRef(() => BillingService))
    private readonly billing: BillingService,
  ) {}
}
```

اكتب الغلاف نفسه في `BillingService` نحو `OrdersService`. لا تندِ الطرف الآخر من داخل `constructor`. وخطّط لحذفه بالوسيط.

## ورقة مشتركة

```ts
import { Module } from '@nestjs/common';
import { MoneyService } from './money.service.js';

@Module({
  providers: [MoneyService],
  exports: [MoneyService],
})
export class MoneyModule {}
```

الميزتان تستوردان `MoneyModule`. الرسم شجرة، والإقلاع لا يحتاج مصنعًا مؤجلًا.

## ماذا تتأكد منه

1. استيراد الجار من ملفه لا من `index.ts`.
2. بعد الوسيط تختفي دورة `imports`.
3. مع `forwardRef`، احذفه من طرف واحد وتأكد أن الإقلاع يفشل، ثم أرجعه. هذا يثبت أن الطرفين لازمان ما دامت الدورة قائمة.
4. لا تجمع `Scope.REQUEST` مع الدورة.
