# forwardRef

## الهدف من هذا الدرس

تأجيل قراءة الصنف الدائري بـ `forwardRef` على الطرفين، للمزودات وللوحدات، من غير اعتبار المسكن تصميمًا نهائيًا.

## المشكلة التي يحلها

عندما تبقى الدورة بعد حذف البرميل واستخراج ما يمكن استخراجه، يفشل الإقلاع لأن metadata الطرف الآخر غير جاهزة لحظة تحليل الـ `constructor`. `forwardRef(() => Other)` يعطي Nest دالة يقرأها عندما يصبح الصنف معرّفًا. طرف واحد لا يكفي في العلاقة المتبادلة التي يوثقها المثال الرسمي: الجهتان تستعملان `@Inject(forwardRef(() => ...))`.

## الفكرة ببساطة

الاستيراد العادي يحفظ الرمز الآن. الدالة تحفظ طريقة الوصول إلى الرمز لاحقًا. الإطار يستدعي الدالة أثناء حل الرسم بعد تحميل الملفات. لذلك يُكتب الغلاف على كل طرف ما زال يحقن الآخر. الترتيب بين الـ `constructor` يبقى غير محدد.

## التعريف التقني

`forwardRef` من `@nestjs/common`. مثال التوثيق لخدمتين:

```ts
import { Inject, Injectable, forwardRef } from '@nestjs/common';
import { CommonService } from './common.service.js';

@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private readonly commonService: CommonService,
  ) {}
}
```

```ts
import { Inject, Injectable, forwardRef } from '@nestjs/common';
import { CatsService } from './cats.service.js';

@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private readonly catsService: CatsService,
  ) {}
}
```

للوحدات، الغلاف على `imports` في الجهتين:

```ts
import { Module, forwardRef } from '@nestjs/common';
import { CatsModule } from '../cats/cats.module.js';

@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

و `CatsModule` يستورد `forwardRef(() => CommonModule)`. الغلاف على الوحدة لا يغني عن الغلاف على الـ `constructor` إن كانت الدورة بين الصنفين أيضًا. عالج مستوى الرسم الذي تدور فيه الأسهم.

بديل موثق على طرف واحد: `ModuleRef` تجلب النسخة لاحقًا بدل حقنها في البناء. ذلك يُفصل في درس إعادة الرسم وفي فصل `ModuleRef`.

## Mental Model

```text
طرف أ: @Inject(forwardRef(() => ب))
طرف ب: @Inject(forwardRef(() => أ))
الإقلاع ينجح
أي constructor أولًا؟ غير محدد
REQUEST + الدورة؟ قد يبقى الاعتماد undefined
```

المسكن يصلح metadata. لا يصنع ترتيبًا.

## أبسط مثال

الدالتان أعلاه هما أصغر مثال يعمل. احذف `forwardRef` من `CommonService` فقط وأعد الإقلاع. العلاقة ناقصة من جهة واحدة فتعود مشكلة الحل. أرجعهما معًا أو أزل الدورة.

## شرح المثال

`@Inject` لازم لأن نوع المعامل وحده لا يحمل الدالة المؤجلة. `forwardRef` يستقبل مصنعًا بلا وسائط يعيد الصنف. استيراد الصنف في رأس الملف ما زال موجودًا للنوع. الغلاف يمنع استخدام الرمز كقيمة حقن مباشرة لحظة قد يكون فيها غير مهيأ. داخل الدوال بعد البناء استخدم `this.commonService` كالمعتاد، وتجنب نداء الطرف الآخر من داخل الـ `constructor` نفسه.

## ماذا يحدث داخليًا؟

1. المزخرف يخزن مصنعًا لا صنفًا خامًا.
2. بعد تقييم الملفات يستدعي Nest المصنع ويحصل على الصنف.
3. يبني الطرفين من غير عقد على الترتيب.
4. إن رمى `constructor` طرفًا نداءً على الآخر، قد يرى كائنًا لم تكتمل حقوله.
5. مع `Scope.REQUEST` تتكرر هذه الهشاشة لكل طلب.
6. دورة `imports` بالغلاف تسمح للوحدتين برؤية المصدَّر. سطح التصدير ما زال واسعًا إن لم تضيّقه.

## مثال عملي

`OrdersModule` و `BillingModule` تتبادلان الاستيراد لأن كل متحكم يستعمل خدمة الآخر. غلف `imports` في الملفين:

```ts
import { Module, forwardRef } from '@nestjs/common';
import { BillingModule } from '../billing/billing.module.js';
import { OrdersService } from './orders.service.js';

@Module({
  imports: [forwardRef(() => BillingModule)],
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

وإن حقنت `BillingService` داخل `OrdersService` والعكس، كرر الغلاف على الـ `constructor` في الملفين. ثم اكتب في تعليق الإزالة: الفوترة يجب أن تستقبل المجموع وسيطًا. المسكن بلا هذه الجملة يُنسخ في الوحدة التالية.

## أخطاء شائعة

- غلاف على طرف واحد في علاقة متبادلة.
- نداء الطرف الآخر داخل `constructor`.
- `forwardRef` حول برميل ما زال يدور في التحميل. أصلح الاستيراد أولًا.
- نسيان الغلاف على `imports` والإبقاء عليه على الخدمة فقط، فتبقى دورة الوحدات.
- إدخال `Scope.REQUEST` على أحد الطرفين.

## متى نستخدمه؟

دورة مفهومة تعذّر فكها في هذا السحب، على الطرفين، مع اختبار إقلاع.

## متى لا نستخدمه؟

قبل فحص البرميل، وقبل محاولة وسيط أو مزود ثالث. ولا كأسلوب لكل وحدتين تتكلمان.

## مقارنة

| الآلية | ماذا تؤجل | الطرفان |
| --- | --- | --- |
| `forwardRef` في `@Inject` | صنف المزود | مطلوبان للدورة المتبادلة |
| `forwardRef` في `imports` | صنف الوحدة | مطلوبان إن كانت دورة وحدات |
| `ModuleRef.get` لاحقًا | الحقن نفسه إلى ما بعد البناء | يكفي طرف واحد يتأخر |

## تمارين

### تمرين سهل

من أين تُستورد `forwardRef`؟

<details>
<summary>الحل</summary>

من `@nestjs/common`.

</details>

### تمرين متوسط

لماذا `@Inject` يحيط `forwardRef` في مثال الخدمتين؟

<details>
<summary>الحل</summary>

لأن المصنع يجب أن يُحفظ في metadata الحقن. نوع TypeScript وحده لا يمرر الدالة المؤجلة إلى الإطار.

</details>

### تمرين متقدم

غلفت `imports` وبقي `constructor` يحقن الصنف مباشرة. أي دورة ما زالت مفتوحة؟

<details>
<summary>الحل</summary>

دورة المزودات. غلاف الوحدة يحل `imports` فقط. طرفا الـ `constructor` يحتاجان `@Inject(forwardRef(() => ...))` أيضًا ما داما يحقنان بعضهما.

</details>

## أسئلة مراجعة

- لماذا الطرفان معًا؟
- ماذا لا يضمنه نجاح الإقلاع؟
- أين يوضع الغلاف للوحدة مقابل الخدمة؟

## الخلاصة

`forwardRef(() => Other)` على الجهتين يؤجل الرمز حتى يكتمل التحميل. لا ترتب عليه أي `constructor` أولًا، ولا تضفه إلى مزود طلب داخل دورة.

## ماذا بعد؟

إعادة رسم حدود الـ Modules حتى تُحذف الدورة بدل تغليفها.

## المصادر الرسمية

- NestJS — Circular dependency: https://docs.nestjs.com/fundamentals/circular-dependency
