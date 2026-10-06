# أمثلة عملية

ثلاثة تسجيلات لمكتب إعارة. كل مثال وصفة مختلفة: صنف، قيمة مع `Symbol`، ومصنع ينتظر خيارًا. الاستيراد النسبي ينتهي بـ `.js`.

## مثال 1 — Service مسجّلة وتُحقن

الغرامة في الخدمة، والمسار يعيد الناتج. النطاق الافتراضي نسخة واحدة.

```ts
import { Controller, Get, Injectable, Module, Query } from '@nestjs/common';

@Injectable()
export class FineService {
  quote(days: number) {
    const billable = Math.max(days - 3, 0);
    return { days, amount: billable * 2 };
  }
}

@Controller('fines')
export class FinesController {
  constructor(private readonly fines: FineService) {}

  @Get('quote')
  quote(@Query('days') days = '1') {
    return this.fines.quote(Number(days));
  }
}

@Module({
  controllers: [FinesController],
  providers: [FineService],
})
export class FinesModule {}
```

`GET /fines/quote?days=5` يستدعي النسخة المسجّلة. `Number` هنا تحويل ظاهر في المعالج لأن Pipe ما زال لاحقًا. `@Injectable` على `FineService` جاهز لليوم الذي تُضاف فيه اعتمادية.

## مثال 2 — `useClass` ثم اسم بديل

الإشعارات تُبنى كصنف بريد، والرمز القديم يشير إلى النسخة نفسها.

```ts
import { Inject, Injectable, Module } from '@nestjs/common';

@Injectable()
export class Notifier {
  send(text: string) {
    return { channel: 'base', text };
  }
}

@Injectable()
export class MailNotifier extends Notifier {
  override send(text: string) {
    return { channel: 'mail', text };
  }
}

@Injectable()
export class HoldsService {
  constructor(
    private readonly notifier: Notifier,
    @Inject('LEGACY_NOTIFIER') private readonly legacy: Notifier,
  ) {}

  placed(isbn: string) {
    const text = `حجز ${isbn}`;
    this.legacy.send(text);
    return this.notifier.send(text);
  }
}

@Module({
  providers: [
    { provide: Notifier, useClass: MailNotifier },
    { provide: 'LEGACY_NOTIFIER', useExisting: Notifier },
    HoldsService,
  ],
})
export class HoldsModule {}
```

`placed` يرسل مرتين عبر `MailNotifier` نفسها. `useClass` يختار الصنف تحت رمز `Notifier`، و `useExisting` لا يبني بريدًا ثانيًا.

## مثال 3 — مصنع لعميل الفهرس

الخيارات صنف، والعميل قيمة يحسبها المصنع. المنطقة اختيارية.

```ts
import { Inject, Injectable, Module } from '@nestjs/common';

@Injectable()
export class CatalogOptions {
  readonly host = 'https://catalog.internal';
}

export const CATALOG = Symbol('CATALOG');

@Injectable()
export class LookupService {
  constructor(@Inject(CATALOG) private readonly catalog: { host: string }) {}

  shelf(code: string) {
    return { code, host: this.catalog.host };
  }
}

@Module({
  providers: [
    CatalogOptions,
    {
      provide: CATALOG,
      useFactory: (options: CatalogOptions, region?: string) => {
        return { host: options.host, region: region ?? 'jeddah' };
      },
      inject: [CatalogOptions, { token: 'CATALOG_REGION', optional: true }],
    },
    LookupService,
  ],
  exports: [CATALOG],
})
export class LookupModule {}
```

`LookupService` تطلب `Symbol` لا واجهة. غياب `'CATALOG_REGION'` يترك المنطقة `jeddah`. تصدير `CATALOG` يشارك الكائن المحسوب مع من يستورد الـ Module.

## ماذا تتأكد منه

- كل صنف في `constructor` له سجل في `providers` أو وصل عبر `imports` و `exports`.
- `useExisting` يشير إلى رمز مسجّل في المصفوفة نفسها أو في سياقها.
- `Symbol` يُستورد من مكان تعريفه ولا يُعاد إنشاؤه في كل ملف.
- `REQUEST` غير مستخدم هنا؛ الخدمات Singleton.
