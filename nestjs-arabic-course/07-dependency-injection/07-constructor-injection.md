# Constructor Injection

## الهدف من هذا الدرس

كيف يعلن المُنشئ الاحتياج، ولماذا يمسح `import type` والواجهة قدرة Nest على الحل.

## المشكلة التي يحلها

الطلب في المُنشئ يعمل ما دام الرمز حاضرًا وقت التشغيل و TypeScript أصدر نوع المعامل. `import type { TaxPolicy }` يمسح الصنف من JavaScript، فتختفي Metadata. الواجهة تُمسح دائمًا ولا تُصدر `design:paramtypes`. الإقلاع يقول إن Nest لا تستطيع حل الاعتماديات رغم أن الملف «مستورد».

## الفكرة ببساطة

```ts
constructor(private readonly tax: TaxPolicy) {}
```

هذا خاصية معامل: `private` ينشئ الحقل ويسند الوسيط. نوع `TaxPolicy` هو ما تبحث عنه الحاوية إذا كان صنفًا أُصدر في Metadata. `@Inject(token)` يستبدل هذا الاستنتاج عندما لا يوجد صنف.

## التعريف التقني

مع `emitDecoratorMetadata` وأي Decorator على الصنف (`@Injectable()` أو `@Controller()`)، يكتب المترجم مصفوفة أنواع المعاملات. Nest تقرأها عند بناء الرسم. عنصر المصفوفة يجب أن يكون قيمة وقت التشغيل، أي دالة الصنف البانية.

تحذير التوثيق: التعليق يجب أن يشير إلى صنف. الواجهة والاسم المستعار للنوع يُمسحان، فيفشل الإقلاع. `import type` لصنف يمسحه أيضًا فتحدث النتيجة نفسها.

```ts
import type { TaxPolicy } from './tax.policy.js';
```

هذا الاستيراد مناسب لوصف يُستخدم في الأنواع فقط. هو غير مناسب لمعامل تريد حقنه إن كان `TaxPolicy` صنفًا. استيراد القيمة يبقي الرمز:

```ts
import { TaxPolicy } from './tax.policy.js';
```

`@Optional()` بجانب المعامل: الغياب يحل إلى `undefined`. `@Inject(TOKEN)` يكتب الرمز الصريح لذلك الموضع ويتقدم على النوع الممسوح.

حقن الخاصية `@Inject(TOKEN) private readonly tax: TaxPolicy` ممكن. التوثيق يفضّل المُنشئ عندما لا يرث الصنف صنفًا آخر، لأن الاحتياج يبقى في التوقيع. حقن الحقل أنسب عندما يصبح `super()` في كل ابن مزعجًا. ذلك الاستثناء لا يلغي القاعدة.

## Mental Model

```text
صنف + استيراد قيمة + @Injectable
  design:paramtypes = [TaxPolicy]
واجهة أو import type
  لا رمز في المصفوفة
@Inject(SYMBOL)
  الرمز الصريح يتقدم على النوع
```

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';
import { TaxPolicy } from './tax.policy.js';
import { VendorGateway } from './vendor.gateway.js';

@Injectable()
export class DraftService {
  constructor(
    private readonly tax: TaxPolicy,
    private readonly vendors: VendorGateway,
  ) {}

  total(net: number) {
    return net * (1 + this.tax.rate());
  }
}
```

## شرح المثال

- الترتيب في المُنشئ هو ترتيب ما تبنيه الحاوية من وسائط، بعد أن تكون الاعتماديات ذاتها قد حُلّت.
- `readonly` يبقي المرجع الذي سلّمته الخانة.
- الصنفان مستوردان كقيم حتى يدخلا Metadata.
- `@Injectable()` هو Decorator الصنف الذي يتيح إصدار الأنواع. على Controller يكفي `@Controller()`.

## ماذا يحدث داخليًا؟

عند الحل تقرأ Nest معاملات `DraftService`. لكل عنصر تأخذ `@Inject` إن وُجد، وإلا تأخذ خانة `design:paramtypes`. ثم تحل ذلك الرمز في السياق. الناتج يُمرَّر إلى `new` الداخلي بالترتيب.

إن كانت الخانة `Object` أو فارغة لأن النوع مُسح، فلا رمز سياسة تبحث عنه. الرسالة تظهر عند الإقلاع لا كـ `undefined` صامت داخل `total`، إلا إذا كان المعامل `@Optional()`.

واجهة بهذا الشكل لا تُصلح الاستيراد:

```ts
export interface TaxPolicy {
  rate(): number;
}
```

حتى `import { TaxPolicy }` غير موجود أصلًا لأن الواجهة ليست تصدير قيمة. الصف المجرد أو `@Inject` مع `Symbol` هو الإصلاح، كما في Dependency Inversion و Injection Tokens.

## مثال عملي

معامل صريح بجانب نوع ممسوح:

```ts
import { Inject, Injectable, Optional } from '@nestjs/common';
import type { Clock } from './clock.js';
import { CLOCK } from './clock.token.js';
import { VendorGateway } from './vendor.gateway.js';

@Injectable()
export class DraftService {
  constructor(
    private readonly vendors: VendorGateway,
    @Optional() @Inject(CLOCK) private readonly clock?: Clock,
  ) {}
}
```

`Clock` واجهة، لذا `import type` صحيح. `CLOCK` يُستورد قيمة لأنه `Symbol`. `VendorGateway` صنف باستيراده العادي. `@Optional()` هنا يخص الساعة فقط: بوابة غائبة ما زالت توقف الإقلاع.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `import type` لصنف في معامل المُنشئ | الرمز ممسوح و `design:paramtypes` بلا مفتاح |
| واجهة كنوع معامل من غير `@Inject` | لا إصدار لـ `design:paramtypes` |
| `@Optional()` على كل المعاملات لتسكيت الإقلاع | الاعتماد الإلزامي يصبح `undefined` عند أول طلب |
| حقن الحقول مع إخفاء الاحتياج عن المُنشئ بلا وراثة | التوقيع لا يروي قصة الصنف |

## متى نستخدمه؟

لكل اعتمادية تديرها الحاوية. المُنشئ مع استيراد قيمة للأصناف، و `@Inject` للرموز الأخرى، و `@Optional()` عند غياب مقبول لرمز واحد.

## متى لا نستخدمه؟

لا تحول وسيط الطلب إلى معامل مُنشئ. `net` في `total(net)` قيمة النداء، وتختلف كل طلب. المُنشئ لما تعيشه النسخة طوال Scope، لا لجسم JSON الحالي.

## مقارنة

| كتابة المعامل | ماذا ترى الحاوية |
| --- | --- |
| صنف مستورد كقيمة | عنصر `design:paramtypes` |
| `import type` لصنف | لا رمز |
| واجهة | لا رمز |
| `@Inject(token)` | الرمز المكتوب حتى لو النوع واجهة |
| `@Optional()` | الغياب مسموح لذلك الموضع |

## تمارين

### تمرين سهل

ماذا ينشئ `private readonly tax` في معامل المُنشئ؟

<details><summary>الحل</summary>

حقلًا على الصنف يُسند له الوسيط الذي تمرره الحاوية، من غير سطر إسناد يدوي.

</details>

### تمرين متوسط

`TaxPolicy` صنف، والخدمة كتبت `import type`. لماذا يفشل الإقلاع؟

<details><summary>الحل</summary>

استيراد النوع يُحذف من JavaScript، فنوع المعامل لا يبقى رمزًا في Metadata. الحاوية لا تجد مفتاح `TaxPolicy`. الإصلاح استيراد قيمة.

</details>

### تمرين متقدم

لماذا `import type { Clock }` مقبول في المثال العملي بينما المورد نفسه مرفوض لصنف `VendorGateway`؟

<details><summary>الحل</summary>

`Clock` واجهة والرمز الفعلي هو `CLOCK` عبر `@Inject`. الواجهة وصف للمترجم فقط. `VendorGateway` صنف ومفتاحه هو الدالة البانية، فيلزم بقاؤها بعد الترجمة.

</details>

## أسئلة مراجعة

- ماذا يجب أن يحتوي `design:paramtypes` حتى يُحل صنف؟
- أي Decorator يتقدم على النوع المستنتج؟
- متى يفضّل التوثيق المُنشئ على حقن الخاصية؟

## الخلاصة

المُنشئ يعلن الرموز، وخاصية المعامل تنشئ الحقل، و Metadata تحمل الأصناف الحاضرة وقت التشغيل. `import type` والواجهة يزيلان المفتاح، و `@Inject` يعيد المفتاح عندما لا يكون صنفًا.

## ماذا بعد؟

هوية المفتاح نفسه عندما تتعدد الأسماء: Tokens.

## المصادر الرسمية

- NestJS — Providers (Dependency injection): https://docs.nestjs.com/providers#dependency-injection
- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
