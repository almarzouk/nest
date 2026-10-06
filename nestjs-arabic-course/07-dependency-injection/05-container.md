# Container

## الهدف من هذا الدرس

وصف حاوية Nest كسجل رموز داخل حدود Module: وصفة، ونسخة مخزنة، وقواعد من يرى الرمز.

## المشكلة التي يحلها

سطر التركيب اليدوي يعرف كل المراجع لأنها متغيرات محلية. بعد IoC لا يوجد متغير `tax` في `main`. الحاوية هي المكان الذي يقابل رمز `TaxPolicy` بالوصفة ثم بالنسخة. من غير هذا السجل يبقى المُنشئ طلبًا بلا طرف يسلّم.

## الفكرة ببساطة

Nest تحمل IoC container تدير علاقات Providers. لكل رمز مرئي في سياق Module خانة: ماذا تفعل عند الحل، وهل توجد نسخة Singleton جاهزة. السياق ليس كيسًا واحدًا لكل ملفات `src`. Module ترى ما سجلته وما صُدّر إليها عبر `imports`.

## التعريف التقني

الخانة تربط الرمز بواحدة من الوصفات التي عرفتها في فصل Providers: `useClass` أو `useValue` أو `useFactory` أو `useExisting`. الاختصار `providers: [DraftService]` خانة رمزها الصنف ووصفتها `useClass` للصنف نفسه.

النسخة المخزنة خاصة بالنطاق. `DEFAULT` يضع كائنًا واحدًا في الخانة بعد أول حل. `REQUEST` لا يضع كائن التطبيق كله في تلك الخانة؛ لكل طلب خريطة حل أقصر عمرًا. تفاصيل العدوى في درس Scope. هنا المهم أن الحاوية تملك العمر لا الملف.

البحث عن الرمز يتم في سياق الصنف الذي يُبنى: Module التي أعلنت المستهلك، زائد ما استوردته. رمز مصدَّر من `AcquisitionModule` مرئي لمن يستوردها. `RegionalTax` غير المصدَّرة ليست في حاوية المستورد حتى لو استطاع TypeScript استيراد الملف.

لا تنشئ الحاوية بنفسك في تطبيق HTTP العادي. `NestFactory.create` تبنيها من الجذر. `ModuleRef` لاستعلام يدوي لاحق في الدورة، خارج هذا الدرس.

## Mental Model

```text
سياق AcquisitionModule
  TaxPolicy    -> useClass FlatTax    -> نسخة
  DraftService -> useClass DraftService -> نسخة تعرف السياسة
سياق ReportsModule
  يرى DraftService إن صُدّرت
  لا يرى TaxPolicy إن لم تُصدَّر
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { DraftService } from './draft.service.js';
import { FlatTax } from './flat-tax.js';
import { TaxPolicy } from './tax.policy.js';

@Module({
  providers: [DraftService, { provide: TaxPolicy, useClass: FlatTax }],
  exports: [DraftService],
})
export class AcquisitionModule {}
```

## شرح المثال

- خانتان في السياق: `DraftService` و `TaxPolicy`.
- `FlatTax` ليست خانة مستقلة. هي هدف `useClass` تحت رمز السياسة.
- `exports` تنشر خانة `DraftService` فقط إلى المستورد.
- مستورد يحقن `TaxPolicy` يفشل الحل لأن الخانة لم تُنشر، رغم أنها موجودة داخل المشتريات.

## ماذا يحدث داخليًا؟

عند الإقلاع تُملأ السجلات من Metadata كل Module مستوردة من الجذر. الخانة تُنشأ قبل أن يُنادى `total`. حل `DraftService` يسأل السياق عن `TaxPolicy`، فتنفَّذ وصفة `FlatTax`، وتُحفظ النسخة تحت رمز السياسة، ثم تُبنى المسودة وتُحفظ تحت رمزها.

طلب ثانٍ لـ `DraftService` في النطاق الافتراضي يعيد المرجع المخزن ولا يعيد `useClass`. لذلك عدّاد على المسودة عدّاد تطبيق داخل هذا السياق، لا عدّاد كل عملية `create` في الاختبارات التي تبني تطبيقًا جديدًا. كل `NestFactory.create` حاوية جديدة.

`NEST_DEBUG` يطبع تفاصيل إضافية عن الحل عند الإقلاع إن تعقّد الرسم. المتغير بيئة تشغيل، لا خيارًا داخل `@Module`.

## مثال عملي

تقرير يستورد المشتريات ويرى المسودة فقط:

```ts
import { Module } from '@nestjs/common';
import { AcquisitionModule } from '../acquisition/acquisition.module.js';
import { ReportsService } from './reports.service.js';

@Module({
  imports: [AcquisitionModule],
  providers: [ReportsService],
})
export class ReportsModule {}
```

`ReportsService` تطلب `DraftService` في المُنشئ فتجد الخانة المنشورة. طلب `TaxPolicy` من التقرير لا يجد خانة في سياقه. الإصلاح الذي يحافظ على الحد هو تصدير السياسة عمدًا، أو إبقاء الحساب داخل `DraftService` حتى لا يحتاج التقرير الرمز الأدنى.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| اعتبار الحاوية خريطة عامة لكل صنف مستورد في TypeScript | الملف المرئي للمترجم قد يكون مخفيًا عن السياق |
| البحث عن النسخة في متغير داخل `main` | المرجع داخل خانة الرمز |
| تصدير كل الخانات احتياطًا | الحد بين Module يختفي |
| توقع أن `FlatTax` تُحقن بالاسم من غير خانة | الرمز المسجّل هو `TaxPolicy` |

## متى نستخدمه؟

في كل تطبيق Nest يعمل بـ `NestFactory.create`. أنت تتعامل معها عبر `providers` و `imports` و `exports` لا عبر مؤشر حاوية في كل خدمة.

## متى لا نستخدمه؟

لا تطلب من الحاوية بناء DTO جسم الطلب. تلك قيمة يدخلها `@Body` إلى الدالة. ولا تخزن حالة الطلب على نسخة `DEFAULT` أملًا أن الحاوية تعزل الطلبات؛ العزل Scope أو وسائط الدالة.

## مقارنة

| الخانة | المحتوى |
| --- | --- |
| رمز + `useClass` | كيفية البناء، ثم النسخة بعد الحل |
| رمز + `useValue` | المرجع الجاهز من لحظة التسجيل |
| رمز غير مصدَّر | مرئي داخل Module المالكة فقط |
| صنف مستورد استيراد TypeScript | لا خانة إلى أن يُسجَّل أو يُستورد مع تصديره |

## تمارين

### تمرين سهل

أين تعيش نسخة `DraftService` بعد الإقلاع الافتراضي؟

<details><summary>الحل</summary>

في خانة رمزها داخل سياق Module التي سجلتها، أو التي تراها عبر التصدير. ليست متغيرًا في `main`.

</details>

### تمرين متوسط

لماذا يفشل حقن `TaxPolicy` في `ReportsService` مع المثالين أعلاه؟

<details><summary>الحل</summary>

`AcquisitionModule` تصدّر `DraftService` فقط. خانة السياسة تبقى داخل سياق المشتريات، والاستيراد لا ينسخ الخانات غير المصدَّرة.

</details>

### تمرين متقدم

ما الفرق بين حاوية `NestFactory.create` في اختبارين منفصلين؟

<details><summary>الحل</summary>

كل إقلاع يبني سجلات ونسخ Singleton خاصة به. عدّاد على الخدمة في الاختبار الأول لا يظهر في الثاني. داخل الإقلاع الواحد الخانة تشارك النسخة لكل من يحقن الرمز في ذلك السياق.

</details>

## أسئلة مراجعة

- ماذا تحفظ الخانة قبل الحل وبعده للنطاق الافتراضي؟
- كيف تصل خانة إلى Module أخرى؟
- هل `FlatTax` خانة في مثال `useClass`؟

## الخلاصة

الحاوية سجل خانات ضمن سياق Module: رمز ووصفة ونسخة حسب Scope. الرؤية تتبع التسجيل والتصدير والاستيراد، لا شجرة ملفات المشروع.

## ماذا بعد؟

الفرق بين ملء الخانة بالوصفة وبين تنفيذها: Registration و Resolution.

## المصادر الرسمية

- NestJS — Providers: https://docs.nestjs.com/providers
- NestJS — DI fundamentals: https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals
