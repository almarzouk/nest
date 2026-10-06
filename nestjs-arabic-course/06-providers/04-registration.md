# Registration داخل Module

## الهدف من هذا الدرس

ربط الرمز بالوصفة داخل `@Module` حتى تستطيع الحاوية إنشاء Provider عند الحل.

## المشكلة التي يحلها

`@Injectable()` على الملف لا يضيف الصنف إلى أي Module. Controller في `LendingModule` يطلب `FeesService`، والإقلاع يفشل لأن سياق تلك الـ Module لا يملك الرمز. التسجيل هو الربط الذي تبحث عنه الحاوية.

## الفكرة ببساطة

`providers: [FeesService]` اختصار. التوثيق يوسّعه إلى `{ provide: FeesService, useClass: FeesService }`. المفتاح `provide` هو الرمز، و `useClass` الصنف الذي يُنشأ. عندما يتطابق الاسمان نستخدم الاختصار.

## التعريف التقني

```ts
@Module({
  controllers: [FeesController],
  providers: [FeesService],
})
export class FeesModule {}
```

Controller في `controllers` حتى تُبنى مساراته، والخدمة في `providers` حتى يُحل `constructor`. خلط القائمتين لا يجعل الخدمة مسارًا ولا يجعل Controller وصفة حقن لمزوّد آخر.

إن احتاجت `OrdersModule` الخدمة، تبقى الخدمة في `providers` الخاصة بـ `FeesModule` وتُضاف إلى `exports`. المستورِد يكتب `imports: [FeesModule]` ولا يعيد إدراج الصنف في `providers` إن كان يريد النسخة نفسها.

إعادة إدراج `FeesService` في `providers` الخاصة بالمستورِد تسجيل ثانٍ ونسخة ثانية ضمن النطاق الافتراضي.

الأشكال `useValue` و `useFactory` و `useExisting` تستبدل `useClass` في الكائن الطويل. دروسها بعد Scope. هذا الدرس يثبت مكان التسجيل ومعنى الاختصار.

## Mental Model

```text
provide: FeesService  ->  الرمز الذي يطلبه constructor
useClass: FeesService ->  ما تبنيه الحاوية لهذا الرمز
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { FeesController } from './fees.controller.js';
import { FeesService } from './fees.service.js';
import { FeeRates } from './fee-rates.js';

@Module({
  controllers: [FeesController],
  providers: [FeeRates, FeesService],
  exports: [FeesService],
})
export class FeesModule {}
```

## شرح المثال

- `FeeRates` مسجّل لأن `FeesService` تحقنه، وهو غير مصدَّر.
- `FeesService` مصدَّر لأن Module أخرى ستطلب الغرامة من غير أن تملك الجدول.
- ترتيب المصفوفة لا يحدد ترتيب الإنشاء. الحاوية تبني الاعتماديات من الأوراق نحو الجذر.
- الملفات مستوردة بلاحقة `.js` لأن المشروع ESM.

## ماذا يحدث داخليًا؟

أثناء الإقلاع تُحوَّل كل خانة في `providers` إلى سجل: رمز ووصفة. الاختصار يصبح `useClass` للصنف نفسه. عند إنشاء `FeesController` تقرأ Nest نوع المعامل، تجد `FeesService` في سجل هذه الـ Module أو في ما استوردته وصُدّر إليها، ثم تنفذ الوصفة.

البحث لا يشمل كل ملفات `src`. Module غير مستوردة من الجذر لا تدخل الرسم، وتسجيلها لا يعمل.

`nest g service fees` غالبًا يضيف الصنف إلى `providers`. راجع الملف بعد الأمر، خصوصًا إن ولّدت الخدمة من مجلد غير مجلد الـ Module.

## مثال عملي

الجذر يركّب الميزة ولا يعيد تسجيل الخدمة:

```ts
import { Module } from '@nestjs/common';
import { FeesModule } from './fees/fees.module.js';
import { OrdersModule } from './orders/orders.module.js';

@Module({
  imports: [FeesModule, OrdersModule],
})
export class AppModule {}
```

```ts
import { Module } from '@nestjs/common';
import { FeesModule } from '../fees/fees.module.js';
import { OrdersService } from './orders.service.js';

@Module({
  imports: [FeesModule],
  providers: [OrdersService],
})
export class OrdersModule {}
```

`OrdersService` تحقن `FeesService` لأن الاستيراد والتصدير وصلا الرمز. `FeeRates` ما زال غير مرئي هنا، وهذا مقصود.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `@Injectable()` من غير خانة `providers` | لا سجل للرمز |
| نسخ الصنف إلى `providers` في كل Module | نسخة Singleton لكل تسجيل |
| وضع الخدمة في `exports` من غير `providers` | التصدير يشير إلى رمز غير مسجّل في هذه الـ Module |
| الاعتماد على ترتيب الأسماء داخل المصفوفة | الترتيب ليس جدول بناء |

## متى نستخدمه؟

كلما أضفت Provider يُحقن داخل الـ Module نفسها أو خارجها. سجّل الاعتماديات في Module المالكة، وصَدّر ما تحتاجه الحدود الخارجية فقط.

## متى لا نستخدمه؟

لإعادة تسجيل خدمة مستوردة ومصدَّرة وأنت تريد النسخة المشتركة. استيراد الـ Module يكفي. التسجيل الثاني نسخة أخرى.

## مقارنة

| الكتابة | المعنى |
| --- | --- |
| `providers: [FeesService]` | الرمز هو الصنف، والوصفة `useClass` للصنف نفسه |
| `{ provide, useClass }` | الرمز والوصفة ظاهران عندما يختلفان لاحقًا |
| `exports: [FeesService]` | الرمز متاح لمن يستورد الـ Module |
| `imports: [FeesModule]` | إدخال الرموز المصدَّرة إلى سياق المستورِد |

## تمارين

### تمرين سهل

ما المعادل الطويل لـ `providers: [FeesService]`؟

<details><summary>الحل</summary>

`{ provide: FeesService, useClass: FeesService }`.

</details>

### تمرين متوسط

`OrdersModule` تستورد `FeesModule` والخدمة غير مصدَّرة. هل يكفي الاستيراد؟

<details><summary>الحل</summary>

لا. `exports` في `FeesModule` يجب أن يحوي `FeesService`. الاستيراد يعرض ما صُدّر فقط.

</details>

### تمرين متقدم

لماذا تسجيل `FeesService` مرة ثانية داخل `OrdersModule` يغيّر سلوك عدّاد داخل الخدمة؟

<details><summary>الحل</summary>

كل تسجيل Singleton نسخة لسياقه. العدّاد على نسخة الطلبات مستقل عن عدّاد نسخة `FeesModule`. الإبقاء على تسجيل واحد مع `exports` يشارك النسخة والعدّاد.

</details>

## أسئلة مراجعة

- ماذا يمثل `provide` في الشكل الطويل؟
- أين يُسجَّل Controller مقابل Service؟
- متى تعيد Module مستورِدة إدراج الصنف في `providers`؟

## الخلاصة

التسجيل يربط رمزًا بوصفة داخل Module. الاختصار `providers: [FeesService]` يعني أن الرمز والصنف واحد، والتصدير يشارك تلك النسخة لا نسخة جديدة.

## ماذا بعد؟

كتابة الطلب في `constructor`: Injection.

## المصادر الرسمية

- NestJS — Provider registration: https://docs.nestjs.com/providers#provider-registration
- NestJS — Standard providers: https://docs.nestjs.com/fundamentals/custom-providers#standard-providers
