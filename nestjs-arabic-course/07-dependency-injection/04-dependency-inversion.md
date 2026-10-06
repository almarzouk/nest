# Dependency Inversion

## الهدف من هذا الدرس

فصل Dependency Inversion عن DI: الأول اتجاه الاعتماد في التصميم، والثاني آلية تمرير الكائن.

## المشكلة التي يحلها

`DraftService` يمكن أن تستلم `RegionalTax` بالحقن وما زالت تعرف سعر المنطقة واسم الصنف التفصيلي. تغيير المورد إلى ضريبة ثابتة يعدّل نوع المُنشئ في الخدمة العالية. المبدأ يقول إن السياسة العليا تعتمد على تجريد، والتفصيل يُربَط في Module.

## الفكرة ببساطة

Dependency Inversion Principle: الوحدات العالية لا تعتمد على الوحدات الدنيا. الاثنتان تعتمدان على تجريد. التجريد لا يعتمد على التفصيل، والتفصيل يعتمد على التجريد.

في Nest التجريد صنف مجرد أو رمز `Symbol`، والتفصيل `RegionalTax` أو `FlatTax`، والربط `provide: TaxPolicy, useClass: RegionalTax`. DI تقدر أن تحقن `RegionalTax` مباشرة من غير هذا الفصل. الحقن حينها موجود، والانقلاب في اتجاه الاعتماد غير موجود.

## التعريف التقني

صنف مجرد يبقى وقت التشغيل فيصلح رمزًا ووصفًا:

```ts
export abstract class TaxPolicy {
  abstract rate(): number;
}
```

`DraftService` تذكر `TaxPolicy` فقط. `RegionalTax` تنفّذ `rate`. Module المشتريات تختار التنفيذ. الخدمة العالية لا تستورد ملف المنطقة.

واجهة TypeScript تصف الشكل وتُمسح، فلا تصلح رمز حاوية وحدها. إن أردت الواجهة وصفًا، أضف `Symbol` كما في درس Injection Tokens. الصنف المجرد هنا يكفي للطرفين.

المبدأ لا يطلب واجهة لكل صنف. `VendorGateway` إن كان لها تنفيذ واحد مستقر، اعتماد `DraftService` عليها مباشرة DI واضحة من غير تجريد إضافي. التجريد عندما يتبدل التفصيل أو عندما يجب ألا تعرف الطبقة العالية اسم المنطقة أو اسم البريد.

## Mental Model

```text
DraftService --> TaxPolicy <-- RegionalTax
                     ^
                     |
                  FlatTax

Module تربط TaxPolicy بأحد السهمين
```

السهمان نحو التجريد. `DraftService` لا يشير إلى `RegionalTax`.

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';
import { TaxPolicy } from './tax.policy.js';

@Injectable()
export class DraftService {
  constructor(private readonly tax: TaxPolicy) {}

  total(net: number) {
    return net * (1 + this.tax.rate());
  }
}
```

```ts
import { Injectable } from '@nestjs/common';
import { TaxPolicy } from './tax.policy.js';

@Injectable()
export class FlatTax extends TaxPolicy {
  rate() {
    return 0.15;
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { DraftService } from './draft.service.js';
import { FlatTax } from './flat-tax.js';
import { TaxPolicy } from './tax.policy.js';

@Module({
  providers: [DraftService, { provide: TaxPolicy, useClass: FlatTax }],
})
export class AcquisitionModule {}
```

## شرح المثال

- `DraftService` لا تستورد `FlatTax`.
- `{ provide: TaxPolicy, useClass: FlatTax }` هو مكان الانقلاب: التفصيل يُستبدل من غير تحرير `total`.
- `FlatTax` تعتمد على التجريد عبر `extends`.
- حذف الربط يفشل الحل لأن `TaxPolicy` مجرد لا يُبنى وحده. هذا متوقع: التجريد ليس تنفيذًا.

## ماذا يحدث داخليًا؟

الحاوية ترى طلب `TaxPolicy` في مُنشئ `DraftService`. السجل يطابق الرمز ويجد `useClass: FlatTax`، فيبني `FlatTax` ويخزنها تحت الرمز `TaxPolicy`. `DraftService` تستلم كائنًا دالته `rate` هي دالة الضريبة الثابتة.

استبدال الاختبار `{ provide: TaxPolicy, useValue: { rate: () => 0 } }` يبقي `DraftService` كما هي. هذا ثمر الانقلاب: الطبقة العالية ثابتة والربط يتغير.

حقن `FlatTax` بالاسم في المُنشئ DI بلا انقلاب. الخدمة العالية صارت تعرف التفصيل، وModule التي تستبدل التنفيذ يجب أن تغيّر التوقيع أو تعيد تسجيل الصنف نفسه.

## مثال عملي

تقرير المكتب يجمع المسودات عبر السياسة، وModule الإنتاج تختار المنطقة:

```ts
import { TaxPolicy } from './tax.policy.js';
import { RegionalTax } from './regional-tax.js';
import { SystemClock } from './system-clock.js';

@Module({
  providers: [
    SystemClock,
    DraftService,
    { provide: TaxPolicy, useClass: RegionalTax },
  ],
})
export class AcquisitionModule {}
```

`RegionalTax` قد تطلب `SystemClock` في مُنشئها. تلك اعتمادية التفصيل، تبقى في طبقة المنطقة. `DraftService` لا تذكر الساعة. إن صعدت الساعة إلى توقيع المسودة، فالتجريد سرّب تفصيلًا.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| تسمية حقن `RegionalTax` باسم Dependency Inversion | الآلية DI، والاتجاه ما زال نحو التفصيل |
| واجهة فقط كرمز `provide` | الواجهة غير موجودة وقت التشغيل |
| تجريد لكل مستودع بلا تنفيذ ثانٍ متوقع | أصناف مجردة لا تغيّر التصميم وتزيد الملفات |
| استيراد `RegionalTax` داخل `DraftService` لنداء ثابت | السهم العالي عاد إلى التفصيل رغم تسجيل `useClass` |

## متى نستخدمه؟

عندما تتبدل السياسة أو البوابة: ضريبة منطقة مقابل ثابتة، بريد مقابل سجل في الاختبار، موردان تحت عقد `rate` أو `send` واحد. الطبقة التي تحسب الإجمالي تبقى على التجريد.

## متى لا نستخدمه؟

عندما يكون الصنف التفصيلي هو العقد المستقر للتطبيق كله ولن يُستبدل. سجّله باختصار `providers: [VendorGateway]` وحقنه باسمه. DI كافية. أضف التجريد عند أول تبديل حقيقي لا قبل ذلك تحسبًا لكل شيء.

## مقارنة

| | DI | Dependency Inversion |
| --- | --- | --- |
| السؤال | من يمرر الكائن؟ | على ماذا يعتمد الصنف العالي؟ |
| بلا الآخر | حقن صنف ملموس | تصميم يعتمد على تجريد وقد يُبنى يدويًا |
| في Nest | `constructor` والحاوية | `provide` التجريد و `useClass` التفصيل |
| الواجهة الممسوحة | لا تصلح رمزًا | لا تكفي تجريدًا للحاوية وحدها |

## تمارين

### تمرين سهل

ماذا يستورد `DraftService` في المثال: `FlatTax` أم `TaxPolicy`؟

<details><summary>الحل</summary>

`TaxPolicy`. `FlatTax` تظهر في Module فقط.

</details>

### تمرين متوسط

هل حقن `VendorGateway` الملموسة في المُنشئ DI؟ وهل هو Dependency Inversion؟

<details><summary>الحل</summary>

DI نعم، لأن الحاوية تمرر البوابة. Dependency Inversion لا، ما دامت الخدمة العالية تسمي الصنف التفصيلي ولا يوجد تجريد تتبدل تحته التنفيذات.

</details>

### تمرين متقدم

لماذا صعود `SystemClock` إلى مُنشئ `DraftService` يضعف التجريد حتى لو بقيت `TaxPolicy`؟

<details><summary>الحل</summary>

الساعة تفصيل لكيفية حساب المنطقة. الطبقة العالية صارت تعرفها، فتغيير الساعة يمس المسودة. الساعة تبقى في مُنشئ `RegionalTax`، والمسودة تطلب `rate` فقط.

</details>

## أسئلة مراجعة

- إلى أين يشير السهم من `DraftService` في الرسم؟
- لماذا لا تكفي واجهة TypeScript رمزًا للتجريد في الحاوية؟
- متى يكون الحقن المباشر لصنف ملموس كافيًا؟

## الخلاصة

Dependency Inversion اتجاه: العالي والدنيا يلتقيان على تجريد، والربط في Module. DI تمرر الكائن وقد تمرر تفصيلًا مباشرة. الصنف المجرد يصلح رمز Nest لأن له وجودًا وقت التشغيل.

## ماذا بعد؟

أين تُحفظ هذه الربوط: Container.

## المصادر الرسمية

- NestJS — Interfaces and abstract classes: https://docs.nestjs.com/fundamentals/custom-providers#interfaces-and-abstract-classes
- NestJS — DI fundamentals: https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals
