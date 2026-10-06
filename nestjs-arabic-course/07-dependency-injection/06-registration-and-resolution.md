# Registration و Resolution

## الهدف من هذا الدرس

فصل لحظتين: Registration يكتب الوصفة تحت الرمز، و Resolution ينفّذ الوصفة وينتج القيمة.

## المشكلة التي يحلها

«الخدمة مسجّلة إذًا النسخة موجودة» خلط يجعل `useFactory` غير المفهوم: الملف حُمّل والعميل لم يُبنَ بعد. العكس أيضًا: مناداة المصنع عند كل قراءة للملف. اللحظتان مختلفتان. التسجيل عند قراءة `@Module`، والحل عندما تحتاج الحاوية القيمة.

## الفكرة ببساطة

Registration يربط الرمز بالوصفة ويضعه في سياق Module. لا يشترط بناء Singleton في السطر نفسه. Resolution يقرأ الوصفة: يعيد قيمة `useValue`، أو يبني `useClass`، أو ينادي `useFactory`، أو يتبع `useExisting`.

للنطاق `DEFAULT` يتم الحل أثناء الإقلاع بعد اكتمال الرسم، وتُخزَّن النسخة. لـ `REQUEST` يتكرر الحل لكل طلب. الوصفة تبقى التسجيل في الحالتين.

## التعريف التقني

التوثيق يسمي الربط في `providers` تسجيلًا: الرمز `CatsService` يرتبط بالصنف. الاختصار يوسَّع إلى `{ provide, useClass }` قبل أن يُحل.

| الوصفة عند الحل | الناتج |
| --- | --- |
| `useValue` | المرجع الذي ثُبّت عند التسجيل |
| `useClass` | نسخة بعد حل مُنشئ ذلك الصنف |
| `useFactory` | قيمة إرجاع الدالة بعد حل `inject`، والوعد يُنتظَر |
| `useExisting` | ناتج حل الرمز الآخر، لا بناء جديد |

تسجيل رمز مرتين في السياق نفسه يعيد كتابة الخانة أو يخلق تعارضًا يجب أن تراه عند الإقلاع، لا نسختين متعاونتين تحت اسم واحد. نسختان متعمدتان تعنيان رمزين.

`@Optional()` لا يسجل شيئًا. هو يغيّر الحل: الرمز الغائب يصبح `undefined` لذلك المعامل.

## Mental Model

```text
Registration
  TaxPolicy -> { useClass: FlatTax }

Resolution عند الإقلاع (DEFAULT)
  حل FlatTax
  خزّن النسخة تحت TaxPolicy
  ابنِ DraftService بها
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { DraftService } from './draft.service.js';
import { FlatTax } from './flat-tax.js';
import { TaxPolicy } from './tax.policy.js';

@Module({
  providers: [
    { provide: TaxPolicy, useClass: FlatTax },
    DraftService,
  ],
})
export class AcquisitionModule {}
```

## شرح المثال

- تقييم هذا الملف يسجل وصفتين. `FlatTax` لم تُستدعَ بعد بمجرد مرور المترجم على الكائن.
- الحل لاحقًا يبني `FlatTax` ثم `DraftService` لأن الثانية تعتمد على الأولى.
- ترتيب المصفوفة تسجيل لا جدول حل. لو كُتبت `DraftService` أولًا فالحل ما زال يبني السياسة قبلها.
- بعد الحل، قراءة الخانة تعيد النسخة المخزنة طوال عمر التطبيق الافتراضي.

## ماذا يحدث داخليًا؟

`NestFactory.create` يقرأ الوحدات ويسجل الخانات، ثم يحل ما يلزم لبناء Controllers و Providers ذات `DEFAULT`. لذلك مصنع `async` يؤخر `listen`: الحل جزء من الإقلاع لا من أول `GET`.

طلب `GET` بعد ذلك لا يعيد Registration. المعالج موجود، و `this.drafts` مرجع الحل القديم.

مع `Scope.REQUEST` يبقى التسجيل مرة، ويتكرر الحل مع كل طلب على ذلك الفرع. تكلفة ذلك في درس Scope: السلسلة كلها تُحل مجددًا.

`useValue` حالة خاصة: القيمة ناتج التسجيل أصلًا، فالحل يعيدها بلا خطوة بناء. هي ما زالت تمر بمرحلة الحل عندما يسأل مستهلك عن الرمز، والناتج المرجع نفسه.

## مثال عملي

منطقة اختيارية لا تسجيل لها:

```ts
import { Injectable, Optional } from '@nestjs/common';

@Injectable()
export class DraftService {
  constructor(@Optional() private readonly region?: RegionCode) {}

  label() {
    return this.region?.code ?? 'unset';
  }
}
```

إن لم توجد وصفة `RegionCode`، التسجيل لم ينسَ خانة بقدر ما الحل سُمح له أن يتخطاها بسبب `@Optional()`. إضافة `{ provide: RegionCode, useValue: { code: 'jeddah' } }` لاحقًا تغيّر الحل من غير تعديل `label`. هذا فرق المرحلتين: الوصفة الجديدة تسجيل، وظهور `jeddah` نتيجة حل تالٍ.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| اعتبار ترتيب `providers` ترتيب `new` | الحل يتبع الاعتماد لا أسطر المصفوفة |
| توقع نداء `useFactory` عند حفظ الملف فقط | النداء عند الحل؛ مع Singleton مرة أثناء الإقلاع |
| تسجيل الرمز نفسه مرتين أملًا في نسختين | خانة واحدة لكل رمز في السياق |
| `@Optional()` كبديل عن `providers` لمزوّد إلزامي عند غيره | المستهلك الآخر ما زال يفشل عند الحل |

## متى نستخدمه؟

Registration عند إضافة قدرة إلى Module. Resolution تتركه للحاوية عند الإقلاع أو لكل طلب حسب Scope. تدخّلك هو كتابة وصفة صحيحة، لا مناداة المصنع من Controller.

## متى لا نستخدمه؟

لا تحل الرمز يدويًا بـ `new` بعد أن سجلته «للتأكد». الحل الثاني نسخة خارج الخانة. استعلام `ModuleRef` لاحقًا عندما يوجد سبب خارج المُنشئ، وهو درس مستقل.

## مقارنة

| | Registration | Resolution |
| --- | --- | --- |
| متى | قراءة Module عند الإقلاع | عند بناء من يحتاج الرمز |
| الناتج | خانة وصفة | قيمة أو نسخة |
| `DEFAULT` | مرة | مرة ثم تخزين |
| `REQUEST` | مرة | لكل طلب على ذلك الفرع |
| خطأ الوصفة | قد يظهر عند قراءة التعريف | يظهر عند تنفيذها ونقص رمز |

## تمارين

### تمرين سهل

هل `useClass: FlatTax` يبني الكائن لحظة كتابة الكائن في الملف؟

<details><summary>الحل</summary>

لا. الكائن وصفة تسجيل. البناء عند الحل.

</details>

### تمرين متوسط

لماذا مصنع `async` على رمز Singleton يؤخر `app.listen`؟

<details><summary>الحل</summary>

الحل جزء من الإقلاع للنطاق الافتراضي. Nest تنتظر الوعد قبل أن تكتمل الشجرة وتقبل الاتصالات.

</details>

### تمرين متقدم

`RegionCode` بلا تسجيل و `@Optional()` على `DraftService` فقط، بينما `LabelPrinter` تطلب `RegionCode` من غير `@Optional()`. ماذا ينجح وماذا يفشل؟

<details><summary>الحل</summary>

التسجيل لا خانة له. حل `DraftService` ينجح و `region` تكون `undefined`. حل `LabelPrinter` يفشل الإقلاع لأن معاملها ما زال إلزاميًا. `@Optional()` قرار حل لكل معامل لا تسجيل عالمي.

</details>

## أسئلة مراجعة

- ماذا يُخزَّن في الخانة قبل أول حل لـ `useClass`؟
- أي الوصفات تعيد مرجعًا من غير بناء؟
- لماذا ترتيب المصفوفة ليس ترتيب الإنشاء؟

## الخلاصة

التسجيل يكتب الوصفة تحت الرمز داخل السياق. الحل ينفّذها وينتج القيمة، مرة للإقلاع في Singleton وكل طلب على فرع `REQUEST`. `@Optional()` يغيّر الحل عند الغياب ولا يكتب خانة.

## ماذا بعد؟

كيف يُعلَن الطلب في المُنشئ حتى يستطيع الحل قراءته: Constructor Injection.

## المصادر الرسمية

- NestJS — DI fundamentals: https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals
- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
