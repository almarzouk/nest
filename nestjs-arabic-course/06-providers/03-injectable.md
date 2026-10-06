# @Injectable

## الهدف من هذا الدرس

فهم `@Injectable()` كإعلان Metadata: الصنف قابل لإدارة الحاوية، وأنواع معاملاته يمكن قراءتها.

## المشكلة التي يحلها

صنف مسجّل في `providers` وعليه `constructor(rates: FeeRates)` قد يفشل الإقلاع برسالة أن Nest لا تستطيع حل الاعتماديات. من غير Decorator على الصنف لا يصدر TypeScript أنواع المعاملات، فالحاوية لا ترى `FeeRates`.

## الفكرة ببساطة

`@Injectable()` يضع Metadata تقول إن Nest IoC container تستطيع إدارة الصنف. أي Decorator على الصنف يجعل المترجم يصدر `design:paramtypes`. على Provider نستخدم `@Injectable` تحديدًا. على Controller يكفي `@Controller` لنفس الإصدار.

## التعريف التقني

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class FeesService {
  constructor(private readonly rates: FeeRates) {}
}
```

`FeeRates` يجب أن يكون صنفًا موجودًا وقت التشغيل، واستيراده استيراد قيمة لا `import type`. الواجهة لا تُصدر في `design:paramtypes`. هذا الفخ يُشرح مع Constructor Injection في فصل Dependency Injection. هنا القاعدة: اعتمد على أصناف، وضع `@Injectable()` على Provider الذي يحقن غيره.

صنف بلا اعتماديات يمكن أن يُنشأ من غير الـ Decorator إن كان في `providers`. التوثيق والممارسة يضعان `@Injectable()` دائمًا حتى لا ينكسر الصنف يوم تُضاف اعتمادية.

`@Injectable({ scope: Scope.REQUEST })` شكل الخيارات. Scope في الدرس التالي. الافتراضي من غير وسيط هو Singleton.

## Mental Model

```text
@Injectable على FeesService
  -> Metadata الصنف
  -> design:paramtypes فيها FeeRates
  -> الحاوية تبحث عن رمز FeeRates
```

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class FeeRates {
  readonly daily = 2;
}
```

```ts
import { Injectable } from '@nestjs/common';
import { FeeRates } from './fee-rates.js';

@Injectable()
export class FeesService {
  constructor(private readonly rates: FeeRates) {}

  quote(days: number) {
    return days * this.rates.daily;
  }
}
```

التسجيل يسرد الاثنين: `providers: [FeeRates, FeesService]`.

## شرح المثال

- `FeeRates` بلا `constructor` مخصص، ومع ذلك عليه `@Injectable()` حتى يبقى أسلوب التسجيل واحدًا.
- `FeesService` تحمل Decorator لأن لها اعتمادية. من غيره تصبح Metadata المعاملات فارغة أو `Object`.
- `private readonly rates` خاصية معامل. الشرح في درس Injection.
- الاستيراد `import { FeeRates } from './fee-rates.js'` يبقي الرمز وقت التشغيل.

## ماذا يحدث داخليًا؟

`emitDecoratorMetadata` في ترجمة TypeScript، ومع Decorator على الصنف، يكتب مصفوفة أنواع المعاملات على الدالة البانية. Nest تقرأها عند الإقلاع. عنصر المصفوفة هو دالة الصنف البانية، وهي الرمز. `@Inject()` يستطيع استبدال هذا الاستنتاج عندما يكون الرمز نصًا أو `Symbol`، وذلك في درس Injection Tokens.

رسالة الفشل المعتادة تبدأ بـ `Nest can't resolve dependencies`. السبب الشائع هنا: نسيان `@Injectable()` على المستهلك، أو نسيان تسجيل `FeeRates`، أو مسح الرمز بـ `import type`.

## مثال عملي

خدمة الإعارة تعتمد على التسعير، والتسعير يعتمد على الجدول. الثلاثة عليهم الـ Decorator:

```ts
import { Injectable } from '@nestjs/common';
import { FeeRates } from './fee-rates.js';
import { FeesService } from './fees.service.js';

@Injectable()
export class LoansService {
  constructor(private readonly fees: FeesService) {}

  open(days: number) {
    return { days, fee: this.fees.quote(days) };
  }
}
```

`LoansService` من غير `@Injectable()` ومع `constructor(fees: FeesService)` لا تملك الحاوية نوع `FeesService` رغم أن الملف يستورد الصنف في أعلى الملف. الاستيراد وحده لا يكتب Metadata المعاملات.

`@Controller()` على `LoansController` يغني عن `@Injectable()` لذلك الصنف لأن Decorator الصنف موجود. لا تضع الاثنين أملًا في سلوك إضافي.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| Provider باعتماديات ومن غير `@Injectable()` | `design:paramtypes` فارغة ويفشل الحل |
| `import type { FeeRates }` ثم حقنه | الرمز ممسوح وقت التشغيل |
| الاعتماد على واجهة في `constructor` | الواجهة لا تظهر في Metadata |
| خلط `@Injectable()` مع تسجيل `providers` | الزينة تعلن قابلية الإدارة، والمصفوفة هي التسجيل |

## متى نستخدمه؟

على كل Provider: Service و Repository وصنف مساعد تسجله الحاوية. وعلى الصنف الذي ستضيف له اعتمادية لاحقًا.

## متى لا نستخدمه؟

على DTO الذي يصف جسم طلب فقط. ذلك صنف بيانات في فصل Body، وليس شيئًا تنشئه الحاوية. وعلى Controller، `@Controller` يؤدي إصدار الأنواع.

## مقارنة

| الإشارة | ماذا تكفي له |
| --- | --- |
| `@Injectable()` | إدارة Provider وإصدار أنواع `constructor` |
| `@Controller()` | خريطة HTTP وإصدار أنواع `constructor` لذلك الصنف |
| `providers: [...]` | إدخال الرمز في السجل |
| وجود الملف | لا Metadata ولا تسجيل |

## تمارين

### تمرين سهل

من أي حزمة يُستورد `Injectable`؟

<details><summary>الحل</summary>

من `@nestjs/common`.

</details>

### تمرين متوسط

`FeesService` تعتمد على `FeeRates` والإقلاع يقول إن الاعتماديات لا تُحل. اذكر سببين متعلقين بهذا الدرس.

<details><summary>الحل</summary>

غياب `@Injectable()` عن `FeesService` فيضيع نوع المعامل، أو `import type` لمزود `FeeRates` فيمسح الرمز. سبب ثالث خارج الزينة: `FeeRates` غير موجودة في `providers`.

</details>

### تمرين متقدم

لماذا Controller يحقن `FeesService` من غير `@Injectable()` على Controller؟

<details><summary>الحل</summary>

`@Controller()` Decorator على الصنف، فيصدر TypeScript `design:paramtypes` لمعاملاته. `@Injectable()` هو الـ Decorator المخصص عندما لا يوجد غيره، كما في Service.

</details>

## أسئلة مراجعة

- ماذا تعلن Metadata الخاصة بـ `@Injectable()`؟
- متى يصبح الـ Decorator ضروريًا لحل `constructor`؟
- هل يغني عن مصفوفة `providers`؟

## الخلاصة

`@Injectable()` يجعل Provider مرئيًا للإدارة ويتيح إصدار أنواع المعاملات. التسجيل في Module يبقى خطوة مستقلة، والرمز المحقون يبقى صنفًا حاضرًا وقت التشغيل.

## ماذا بعد؟

وضع الصنف في مصفوفة `providers`: Registration داخل Module.

## المصادر الرسمية

- NestJS — Providers: https://docs.nestjs.com/providers
- NestJS — Custom providers (DI fundamentals): https://docs.nestjs.com/fundamentals/custom-providers
