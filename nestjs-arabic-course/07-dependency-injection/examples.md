# أمثلة عملية

ثلاثة رسوم لمكتب المشتريات. كل مثال يبرز قرارًا واحدًا: إخراج `new`، الربط على تجريد، ومشاركة Singleton عبر التصدير. الاستيراد النسبي ينتهي بـ `.js`.

## مثال 1 — التركيب في الحاوية لا داخل الخدمة

الساعة والسياسة والمسودة أوراق ثم جذر. Controller يعيد الناتج ولا يبني أحدًا.

```ts
import { Controller, Get, Injectable, Module, Query } from '@nestjs/common';

@Injectable()
export class SystemClock {
  now() {
    return new Date();
  }
}

@Injectable()
export class TaxPolicy {
  constructor(private readonly clock: SystemClock) {}

  rate() {
    return this.clock.now().getMonth() === 11 ? 0.05 : 0.15;
  }
}

@Injectable()
export class DraftService {
  constructor(private readonly tax: TaxPolicy) {}

  total(net: number) {
    return { gross: net * (1 + this.tax.rate()) };
  }
}

@Controller('drafts')
export class DraftsController {
  constructor(private readonly drafts: DraftService) {}

  @Get('preview')
  preview(@Query('net') net = '100') {
    return this.drafts.total(Number(net));
  }
}

@Module({
  controllers: [DraftsController],
  providers: [SystemClock, TaxPolicy, DraftService],
})
export class AcquisitionModule {}
```

الحل أثناء الإقلاع: الساعة، ثم السياسة، ثم المسودة، ثم Controller. `GET /drafts/preview?net=200` ينادي `total` على النسخة نفسها لكل الطلبات. `Number` تحويل ظاهر لأن Pipe لاحق.

## مثال 2 — الانقلاب على سياسة قابلة للاستبدال

المسودة تعرف التجريد. الاختبار يربط `useValue` من غير تحرير `total`.

```ts
import { Injectable, Module } from '@nestjs/common';

export abstract class TaxPolicy {
  abstract rate(): number;
}

@Injectable()
export class FlatTax extends TaxPolicy {
  rate() {
    return 0.15;
  }
}

@Injectable()
export class DraftService {
  constructor(private readonly tax: TaxPolicy) {}

  total(net: number) {
    return net * (1 + this.tax.rate());
  }
}

@Module({
  providers: [DraftService, { provide: TaxPolicy, useClass: FlatTax }],
})
export class AcquisitionModule {}

@Module({
  providers: [
    DraftService,
    { provide: TaxPolicy, useValue: { rate: () => 0 } },
  ],
})
export class AcquisitionTestModule {}
```

`import type` لـ `TaxPolicy` هنا يكسر الحل لأن الصنف المجرد هو المفتاح. استيراده في ملف المسودة يكون استيراد قيمة. الواجهة الممسوحة لا تحل محل `abstract class`.

## مثال 3 — عقدة مصدَّرة وعدّاد مشترك

التقرير يرى المسودة فقط. التسجيل الثاني للساعة كان سينسخ الورقة، وهو غير موجود هنا عمدًا.

```ts
import { Injectable, Module } from '@nestjs/common';
import { DraftService } from './draft.service.js';
import { SystemClock } from './system-clock.js';
import { TaxPolicy } from './tax.policy.js';

@Module({
  providers: [SystemClock, TaxPolicy, DraftService],
  exports: [DraftService],
})
export class AcquisitionModule {}

@Injectable()
export class ReportsService {
  constructor(private readonly drafts: DraftService) {}

  previewPair() {
    return [this.drafts.total(10), this.drafts.total(10)];
  }
}

@Module({
  imports: [AcquisitionModule],
  providers: [ReportsService],
})
export class ReportsModule {}
```

`ReportsService` تحقن الرمز المصدَّر فتشارك Singleton المشتريات. `previewPair` ينادي النسخة نفسها مرتين. `TaxPolicy` و `SystemClock` غير مصدَّرتين، فرسم التقارير يتوقف عند المسودة. تسجيل الساعة مرة ثانية داخل `ReportsModule` كان سينسخ الورقة، وهو غير موجود هنا عمدًا.

## ماذا تتأكد منه

- لا `new DraftService` داخل المعالج.
- مفتاح المُنشئ صنف مستورد كقيمة، أو `@Inject` لرمز غير صنفي.
- كل سهم في الرسم له خانة في السياق المرئي.
- حالة الطلب ليست حقلًا على نسخة `DEFAULT`.
