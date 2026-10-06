# Injection

## الهدف من هذا الدرس

كتابة اعتمادية في `constructor` حتى تمرر الحاوية النسخة، مع خاصية المعامل `@Optional()`.

## المشكلة التي يحلها

`this.fees = new FeesService()` داخل Controller يخفي الاعتمادية عن التسجيل ويصنع نسخة خارج السجل. الحقن يعلن الاحتياج في التوقيع، وNest تملأ الوسيط قبل أن يعمل أي مسار.

## الفكرة ببساطة

```ts
constructor(private readonly fees: FeesService) {}
```

السطر خاصية معامل: `private` ينشئ الحقل ويسند الوسيط، فلا تكتب `this.fees = fees`. نوع `FeesService` هو الرمز الذي تبحث عنه الحاوية، بشرط أن TypeScript أصدره في Metadata.

## التعريف التقني

الحقن بالمُنشئ هو الأسلوب الافتراضي. Nest تقرأ أنواع المعاملات وتمرر النسخ. النطاق الافتراضي يمرر النسخة المشتركة.

`@Optional()` على معامل يجعل الغياب `undefined` بدل فشل الإقلاع. الصنف يوفّر البديل:

```ts
import { Injectable, Optional } from '@nestjs/common';

@Injectable()
export class FeesService {
  constructor(@Optional() rates?: FeeRates) {
    this.daily = rates?.daily ?? 2;
  }

  private readonly daily: number;
}
```

من غير `@Optional()` غياب `FeeRates` من السجل يوقف الإقلاع. الـ Decorator لا يصنع تسجيلًا؛ هو يغيّر سلوك الرمز الناقص فقط.

حقن الخاصية بـ `@Inject()` على حقل ممكن عندما ترث صنفًا وتمرير `super()` يصبح مزعجًا. إن كان الصنف لا يرث، المُنشئ أوضح لأن الاحتياج ظاهر في التوقيع. تفاصيل `@Inject` للرموز غير الصنفية في درس Injection Tokens.

لا تستخدم `import type` لصنف تحقنه. ذلك يمسح الرمز و Metadata. الواجهة لا تصلح رمزًا. الفصل التالي يفتح السبب؛ هنا التزم بصنف مستورد استيراد قيمة.

## Mental Model

```text
إنشاء FeesController
  اقرأ معامل constructor
  جد رمز FeesService في السجل
  مرر النسخة إلى الخاصية fees
```

## أبسط مثال

```ts
import { Controller, Get } from '@nestjs/common';
import { FeesService } from './fees.service.js';

@Controller('fees')
export class FeesController {
  constructor(private readonly fees: FeesService) {}

  @Get('daily')
  daily() {
    return { amount: this.fees.quote(1) };
  }
}
```

## شرح المثال

- `readonly` يمنع استبدال الحقل بعد البناء. النسخة تبقى التي مرتها الحاوية.
- `quote(1)` استدعاء عادي. الحقن تم قبل أول طلب.
- `@Controller` يكفي لإصدار أنواع معاملات Controller. الخدمة نفسها عليها `@Injectable()` ومسجّلة.
- `GET /fees/daily` يعيد كائنًا برمز 200.

## ماذا يحدث داخليًا؟

الحاوية لا تنادي `new FeesController()` من كودك. هي تجمع وسائط المُنشئ من السجل ثم تستدعيه. إن كانت `FeesService` تعتمد على `FeeRates`، تُحل `FeeRates` أولًا. هذا البناء من الأوراق نحو الجذر يُفصَّل في Dependency Graph.

حقن نفس الرمز في Controller و Service ضمن التسجيل الواحد يعيد النسخة Singleton نفسها. مقارنة `===` بينهما ناجحة عند النطاق الافتراضي.

## مثال عملي

خدمة الطلبات تطلب التسعير، والتسعير اختياري الجدول:

```ts
import { Injectable, Optional } from '@nestjs/common';
import { FeeRates } from './fee-rates.js';
import { FeesService } from './fees.service.js';

@Injectable()
export class OrdersService {
  constructor(
    private readonly fees: FeesService,
    @Optional() private readonly rates?: FeeRates,
  ) {}

  preview(days: number) {
    return {
      amount: this.fees.quote(days),
      daily: this.rates?.daily ?? null,
    };
  }
}
```

`FeesService` إلزامية: غيابها فشل إقلاع. `FeeRates` اختيارية في هذا الصنف فقط. إن كانت الخدمة نفسها تطلب `FeeRates` من غير `@Optional()`، فالتسجيل ما زال مطلوبًا لتلك الخدمة. `@Optional()` قرار كل مستهلك على حدة.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `constructor() { this.fees = new FeesService(); }` | نسخة خارج السجل، واعتماديات الخدمة لا تُحل |
| `import type { FeesService }` في ملف الحقن | الرمز ممسوح ويفشل الحل |
| `@Optional()` بدل تسجيل مزوّد إلزامي | الحقل `undefined` والبديل يعمل بصمت |
| حقن صنف غير مصدَّر من Module أخرى | الرمز غير مرئي في سياق المستهلك |

## متى نستخدمه؟

كلما احتاج صنف تديره Nest إلى Provider مسجّل. المُنشئ هو المكان. `@Optional()` عندما للميزة سلوك واضح من غير ذلك الرمز.

## متى لا نستخدمه؟

لإنشاء DTO أو كائن طلب داخل الدالة. تلك قيم تسجّل بـ `new` العادي لأنها ليست في السجل. ولاختيار تنفيذ مختلف حسب البيئة؛ ذلك `useClass` في التسجيل لا `if` داخل المُنشئ.

## مقارنة

| الأسلوب | من ينشئ الاعتمادية |
| --- | --- |
| معامل `constructor` | الحاوية، قبل أول استخدام |
| `new` داخل الدالة | الدالة، في كل مسار كتبته |
| `@Optional()` | الحاوية تمرر النسخة أو `undefined` |
| حقن حقل بـ `@Inject()` | الحاوية، أنسب عند الوراثة المزعجة |

## تمارين

### تمرين سهل

ماذا يوفّر `private readonly fees` في معامل المُنشئ؟

<details><summary>الحل</summary>

حقلًا على الصنف مُسندًا له وسيط المُنشئ، من غير سطر `this.fees = fees`.

</details>

### تمرين متوسط

متى تنجح مقارنة `controller.fees === service.fees` للنطاق الافتراضي؟

<details><summary>الحل</summary>

عندما يحقن الطرفان رمز `FeesService` من التسجيل نفسه. Singleton يخزن نسخة واحدة ويمررها لكل المستهلكين في ذلك السياق.

</details>

### تمرين متقدم

`OrdersService` تعلّم `FeeRates` بـ `@Optional()`، و `FeesService` تطلب `FeeRates` من غير ذلك. هل يكفي حذف `FeeRates` من `providers`؟

<details><summary>الحل</summary>

لا. `@Optional()` يؤثر في `OrdersService` فقط. `FeesService` ما زالت تطلب الرمز، وغيابه يوقف الإقلاع عند حلها.

</details>

## أسئلة مراجعة

- ماذا يمثل نوع معامل المُنشئ للحاوية؟
- ماذا يحقن `@Optional()` عند غياب التسجيل؟
- متى تفضّل المُنشئ على حقن الحقل؟

## الخلاصة

المُنشئ يعلن الرموز، وخاصية المعامل تنشئ الحقل، والحاوية تمرر النسخ. `@Optional()` يحوّل الغياب إلى `undefined` لذلك المعامل فقط.

## ماذا بعد؟

عمر النسخة: Scope بنظرة أولى.

## المصادر الرسمية

- NestJS — Providers (Dependency injection): https://docs.nestjs.com/providers#dependency-injection
- NestJS — Optional providers: https://docs.nestjs.com/providers#optional-providers
