# Value Providers

## الهدف من هذا الدرس

تسجيل قيمة جاهزة بـ `useValue`: ثابت، كائن مكتبة أُنشئ خارج الحاوية، أو مزيف يطابق شكل صنف.

## المشكلة التي يحلها

جدول أسعار الشحن ليس صنفًا يحتاج `new` في كل إقلاع. ومزيف الاختبار قد يكون كائنًا حرفيًا فيه `send`. `useClass` ما زال يطلب دالة بانية. `useValue` يضع القيمة تحت الرمز كما هي.

## الفكرة ببساطة

```ts
{
  provide: ShippingRates,
  useValue: { standard: 15, express: 40 },
}
```

من يطلب الرمز يحصل على هذا الكائن. الحاوية لا تستدعي بناءً ولا تقرأ `design:paramtypes` للقيمة.

## التعريف التقني

`useValue` يناسب ثابتًا، أو مكتبة خارجية أنشأتها أنت، أو استبدال خدمة بكائن مزيف في الاختبار. التوثيق يذكر أن structural typing في TypeScript يقبل أي كائن يطابق الدوال المطلوبة، بما فيه كائن حرفي أو نسخة صنعتها بـ `new` قبل التسجيل.

القيمة تُلتقط عند تحميل Module. الكائن نفسه هو ما يُمرَّر لكل المستهلكين عند Scope الافتراضي. تعديل حقل عليه ظاهر لكل من حقن الرمز.

لا تجمع `useValue` مع `useClass` في السجل نفسه. مفتاح واحد يحدد الوصفة.

رمز نصي مثل `'SHIPPING_RATES'` صالح هنا، وحقنه يتم بـ `@Inject`. درس Injection Tokens يفصل الرموز غير الصنفية. هذا الدرس يركز على أن الوصفة قيمة لا بناء.

تصدير القيمة يكون بالرمز:

```ts
exports: [ShippingRates]
```

## Mental Model

```text
useValue: ratesObject
    -> السجل يحتفظ بالمرجع
    -> constructor يحصل على المرجع نفسه
```

## أبسط مثال

```ts
import { Injectable, Module } from '@nestjs/common';

export class ShippingRates {
  standard: number;
  express: number;
}

export const shippingRatesProvider = {
  provide: ShippingRates,
  useValue: { standard: 15, express: 40 },
};

@Injectable()
export class ShippingService {
  constructor(private readonly rates: ShippingRates) {}

  price(speed: 'standard' | 'express') {
    return this.rates[speed];
  }
}

@Module({
  providers: [shippingRatesProvider, ShippingService],
  exports: [ShippingRates],
})
export class ShippingModule {}
```

## شرح المثال

- `ShippingRates` صنف رمز. `useValue` لا ينادي `new ShippingRates()`.
- الكائن الحرفي يكفي لأن الخدمة تقرأ `standard` و `express`.
- `price` تعيد رقمًا من الجدول المشترك.
- `exports` فيها الرمز `ShippingRates`، فيحقنه من يستورد `ShippingModule`.

## ماذا يحدث داخليًا؟

عند الحل تعيد الحاوية القيمة المخزنة للرمز. لا توجد سلسلة اعتماديات داخل الوصفة؛ القيمة جاهزة حتى لو اعتمد مستهلكوها على غيرها. إن كانت القيمة نسخة أنشأتها بـ `new Client(url)` قبل `@Module`، فأنت المسؤول عن وسائط `Client`، لا الحاوية.

في الاختبار:

```ts
{
  provide: ShippingService,
  useValue: { price: () => 0 },
}
```

أي مستهلك يطلب `ShippingService` يحصل على هذا الكائن. الدوال الناقصة تظهر عند الاستدعاء، لذلك المزيف يطابق ما يستدعيه الاختبار فعلًا.

## مثال عملي

عميل فهرس أُنشئ في ملف تهيئة ويُسجَّل كما هو:

```ts
import { Module } from '@nestjs/common';
import { CatalogClient } from './catalog-client.js';
import { CatalogService } from './catalog.service.js';

const client = new CatalogClient('https://catalog.internal');

@Module({
  providers: [
    CatalogService,
    { provide: CatalogClient, useValue: client },
  ],
})
export class CatalogModule {}
```

`CatalogService` تطلب `CatalogClient` في المُنشئ. الحاوية تمرر `client` الذي يتصل بالعنوان المكتوب عند تحميل الملف. عنوان يعتمد على Provider آخر يُبنى داخل `useFactory` لا داخل `useValue`.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `useValue: ShippingRates` تمرير الصنف لا الكائن | المستهلك يستلم الدالة البانية بدل الجدول |
| تعديل `rates.standard` من طلب | كل الطلبات ترى السعر الجديد لأن المرجع واحد |
| مزيف ينقصه دالة يستدعيها المسار | الفشل يظهر عند الطلب لا عند الإقلاع |
| توقع حل اعتماديات داخل كائن `useValue` | القيمة لا تُبنى من الحاوية |

## متى نستخدمه؟

لجدول ثابت، أو مزيف اختبار يطابق الدوال، أو كائن مكتبة صنعته قبل الإقلاع وتعرف وسائطه.

## متى لا نستخدمه؟

عندما يجب أن تبني الحاوية الصنف وتحقن وسائطه. ذلك `useClass`. وعندما تُحسب القيمة من مزودات أخرى. ذلك `useFactory`.

## مقارنة

| الحاجة | الوصفة |
| --- | --- |
| `new` تديره Nest لصنف تختاره | `useClass` |
| مرجع جاهز أو ثابت | `useValue` |
| حساب يعتمد على مزودات | `useFactory` |
| اسم ثانٍ للنسخة المسجّلة | `useExisting` |

## تمارين

### تمرين سهل

هل تستدعي الحاوية بناء `ShippingRates` في مثال `useValue`؟

<details><summary>الحل</summary>

لا. تمرر الكائن `{ standard: 15, express: 40 }` كما كُتب.

</details>

### تمرين متوسط

كيف تصدّر الجدول إلى Module أخرى؟

<details><summary>الحل</summary>

`exports: [ShippingRates]` باستخدام الرمز الذي في `provide`. المستورِد يحقن `ShippingRates`.

</details>

### تمرين متقدم

لماذا `useValue: { price: () => 0 }` بدل `useClass: FakeShipping` في اختبار ضيق؟

<details><summary>الحل</summary>

الاختبار يحتاج دالة `price` فقط. الكائن يطابق هذا الاستدعاء من غير ملف صنف. إن كان للمزيف اعتماديات تريد Nest أن تبنيها، انتقل إلى `useClass`.

</details>

## أسئلة مراجعة

- ماذا تخزن الحاوية تحت الرمز مع `useValue`؟
- لماذا تغيير حقل القيمة ظاهر لكل المستهلكين؟
- ماذا يحدث إن وُضع الصنف نفسه في `useValue` بدل نسخة أو كائن؟

## الخلاصة

`useValue` يسجل مرجعًا جاهزًا تحت الرمز. لا بناء ولا حقن وسائط داخل القيمة. الثابت والمزيف والعميل المُنشأ خارجًا هي الحالات المناسبة.

## ماذا بعد؟

حساب القيمة من مزودات أخرى: Factory Providers.

## المصادر الرسمية

- NestJS — Value providers: https://docs.nestjs.com/fundamentals/custom-providers#value-providers-usevalue
