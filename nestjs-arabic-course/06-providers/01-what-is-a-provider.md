# ما هو Provider

## الهدف من هذا الدرس

تعريف Provider: أي شيء تسجّله الحاوية ليُحقَن في صنف آخر.

## المشكلة التي يحلها

Controller الذي ينشئ مستودعًا وأداة تسعير بنفسه يملك شبكة كائنات خفية. Provider يخرج هذه الكائنات إلى تسجيل صريح، وNest توصلها لمن يطلبها.

## الفكرة ببساطة

التوثيق يقول إن Services و Repositories و Factories و Helpers يمكن أن تكون Providers. الفكرة أن الصنف يُحقن كاعتمادية، فتتكوّن علاقات بين الكائنات، ووقت التشغيل يتولى التوصيل.

في أبسط شكل، Provider صنف JavaScript داخل مصفوفة `providers` في Module.

## التعريف التقني

التسجيل الافتراضي هو اسم الصنف:

```ts
providers: [FeesService]
```

هذا اختصار لربط الرمز `FeesService` بالصنف نفسه. الأشكال `useClass` و `useValue` و `useFactory` و `useExisting` دروس لاحقة في هذا الفصل. هنا يكفي أن الحاوية لا ترى إلا ما سُجّل.

Controller يبقى في `controllers`. هو يستهلك Provider ولا يُعاد إعلانه داخل `providers` في الاستعمال المعتاد.

`@Injectable()` على الصنف يعلن أنه قابل للإدارة. تفصيل الزينة في درسها. النطاق الافتراضي Singleton: نسخة واحدة تُشارك. Scope درس مستقل.

## Mental Model

```text
Module.providers  ->  سجل الحاوية
        |
        v
Controller أو Provider آخر يطلب النوع في constructor
        |
        v
النسخة نفسها تُمرَّر عند النطاق الافتراضي
```

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class FeesService {
  lateFee(days: number) {
    return days * 2;
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { FeesService } from './fees.service.js';

@Module({
  providers: [FeesService],
})
export class FeesModule {}
```

## شرح المثال

- الصنف قاعدة غرامة بسيطة. لا مسار HTTP هنا.
- `@Injectable()` من `@nestjs/common` يضع Metadata الإدارة.
- `providers: [FeesService]` هو التسجيل. من غيره لا تملك الحاوية رمزًا تمرره.
- `FeesModule` حد التسجيل، كما في فصل Modules. التصدير لازم فقط عندما تحقنه Module أخرى.

## ماذا يحدث داخليًا؟

عند الإقلاع تقرأ Nest مصفوفة `providers` وتسجل الرمز. عندما تبني صنفًا يعتمد على `FeesService` تبحث عن الرمز، وتنشئ النسخة مرة ضمن النطاق الافتراضي، وتخزنها، وتمررها إلى `constructor`. إنشاء الاعتماديات بـ `new` داخل الملفات موضوع فصل Dependency Injection. هنا التسجيل هو الباب.

Provider يمكن أن يكون قيمة أو مصنعًا، لا صنف خدمة فقط. تلك الأشكال تأتي بعد أن يثبت تسجيل الصنف.

## مثال عملي

مستودع وقارئة أسعار بجانب الخدمة، والخدمة هي التي تُصدَّر:

```ts
import { Module } from '@nestjs/common';
import { FeeRates } from './fee-rates.js';
import { FeesService } from './fees.service.js';
import { LoansRepository } from './loans.repository.js';

@Module({
  providers: [FeesService, FeeRates, LoansRepository],
  exports: [FeesService],
})
export class FeesModule {}
```

`FeeRates` و `LoansRepository` مرئيان داخل `FeesModule` فقط. Module الطلبات التي تستورد `FeesModule` تستطيع حقن `FeesService` لأن اسمها في `exports`. الحقن نفسه يُكتب في `constructor`، ودرسه يلي التسجيل.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| ملف `fees.service.ts` من غير `providers` | الحاوية لا تعرف الرمز رغم أن TypeScript يستورد الملف |
| وضع `FeesService` في `controllers` | الخدمة تُعامل كمدخل HTTP |
| توقع أن كل صنف في المجلد Provider | التسجيل قرار في `@Module` |
| `new FeesService()` داخل Controller بجانب التسجيل | نسختان: واحدة من الحاوية وأخرى محلية |

## متى نستخدمه؟

لكل قدرة يريد صنف آخر استخدامها عبر الحاوية: حساب، مستودع، عميل HTTP، إعداد ثابت ستحقنه لاحقًا كقيمة.

## متى لا نستخدمه؟

لدالة تُستدعى مرة داخل ملف واحد ولا يحتاجها صنف آخر. تلك دالة عادية في الملف. ولصنف مسارات HTTP. ذلك Controller في `controllers`.

## مقارنة

| المكان | الدور |
| --- | --- |
| `providers` | ما تنشئه الحاوية أو تحفظه تحت رمز |
| `controllers` | مداخل HTTP |
| `exports` | ما يجوز لـ Module مستورِدة أن تحقنه |
| استيراد TypeScript | يجعل الاسم ظاهرًا للمترجم، ولا يسجل الرمز |

## تمارين

### تمرين سهل

اذكر ثلاثة أشكال يصنفها التوثيق كـ Provider.

<details><summary>الحل</summary>

من القائمة: Service و Repository و Factory و Helper. أي منها يُسجَّل ليُحقن.

</details>

### تمرين متوسط

`FeesService` مستوردة في Controller و `FeesModule` لا تسجلها. ماذا ينقص الحاوية؟

<details><summary>الحل</summary>

سجلًا في `providers`، ثم تصديرًا إن كان Controller في Module أخرى. استيراد الملف لا يغني عن المصفوفة.

</details>

### تمرين متقدم

لماذا يبقى `FeeRates` خارج `exports` بينما الخدمة داخله؟

<details><summary>الحل</summary>

جدول الأسعار تفصيل داخل الميزة. `FeesService` هي العقد الذي تستهلكه الطلبات. إخفاء `FeeRates` يمنع Module أخرى من الاعتماد على شكل الجدول.

</details>

## أسئلة مراجعة

- ما الفكرة الرئيسية وراء Provider؟
- أين يُسجَّل صنف حتى تستطيع الحاوية حقنه؟
- ما النطاق الافتراضي الذي ستفصّله لاحقًا؟

## الخلاصة

Provider كائن تديره الحاوية وتوصله لمن يطلبه. التسجيل في `providers` هو ما يجعله مرئيًا، والصنف العادي على القرص لا يدخل السجل وحده.

## ماذا بعد؟

Service كمكان قاعدة العمل الذي يستدعيه Controller.

## المصادر الرسمية

- NestJS — Providers: https://docs.nestjs.com/providers
