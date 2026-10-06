# @Get و @Post و @Put و @Patch و @Delete

## الهدف من هذا الدرس

اختيار Decorator الطريقة الذي يطابق فعل HTTP على المورد، ومعرفة بقية الطرق التي يوثّقها Nest.

## المشكلة التي يحلها

مسار واحد باسم `/editions/save` يخفي هل الطلب ينشئ أو يستبدل. طريقة HTTP هي الفعل، والمسار هو المورد. الخلط بينهما يجعل العميل والذاكرة الوسيطة يعاملان الكتابة كقراءة.

## الفكرة ببساطة

كل Decorator طريقة يسجل معالجًا لفعل واحد. `@Get` يقرأ، `@Post` ينشئ، `@Put` يستبدل، `@Patch` يعدّل جزءًا، `@Delete` يحذف. Nest توفر أيضًا `@Head` و `@Options` و `@All` و `@QueryMethod`.

## التعريف التقني

التوثيق يسرد `@Get()` و `@Post()` و `@Put()` و `@Delete()` و `@Patch()` و `@Options()` و `@Head()` و `@QueryMethod()`. الأخير يربط الطريقة `QUERY`، واسمه `QueryMethod` حتى لا يصطدم بـ `@Query()` الذي يقرأ معاملات الاستعلام.

`@All()` يسجل المسار لكل هذه الطرق.

الطريقة والمسار معًا يحددان نقطة النهاية. `@Post('reprint')` تحت `@Controller('editions')` هو `POST /editions/reprint`.

رمز النجاح الافتراضي 200، و `@Post` يستخدم 201. تغيير الرمز بـ `@HttpCode` درس لاحق في هذا الفصل. هنا نختار الفعل.

## Mental Model

```text
GET    /editions          قراءة القائمة
POST   /editions          إنشاء طبعة
PUT    /editions/current  استبدال الطبعة الحالية
PATCH  /editions/current  تعديل حقول مختارة
DELETE /editions/current  حذف الطبعة الحالية
```

`PUT` و `DELETE` في عقد HTTP أفعال يمكن تكرارها بأثر واحد. `POST` كل استدعاء قد ينشئ موردًا آخر. اختر Decorator وفق هذا العقد.

## أبسط مثال

```ts
import { Controller, Delete, Get, Patch, Post, Put } from '@nestjs/common';
import { EditionsService } from './editions.service.js';

@Controller('editions')
export class EditionsController {
  constructor(private readonly editions: EditionsService) {}

  @Get()
  findAll() {
    return this.editions.findAll();
  }

  @Post()
  create() {
    return this.editions.createDraft();
  }

  @Put('current')
  replace() {
    return this.editions.replaceCurrent();
  }

  @Patch('current')
  retitle() {
    return this.editions.retitleCurrent();
  }

  @Delete('current')
  remove() {
    return this.editions.removeCurrent();
  }
}
```

## شرح المثال

- خمسة أفعال على مورد الطبعات. القطعة `current` ثابتة حتى يبقى الدرس على الطريقة، ومعاملات المسار في الدرس التالي.
- `create` تعيد الطبعة الجديدة. Nest ترسلها برمز 201 لأن الطريقة POST.
- `findAll` و `replace` و `retitle` و `remove` نجاحها 200 ما لم تغيّر الرمز لاحقًا.
- الخدمة تملك معنى «استبدال» و«تعديل العنوان». Controller يعلن الفعل فقط.

## ماذا يحدث داخليًا؟

كل Decorator يضيف الطريقة إلى مفتاح الخريطة مع المسار المدمج. طلب `POST /editions` لا يدخل `findAll` حتى لو كان المسار النصي واحدًا. `@All('current')` يوسع المفتاح لكل الطرق، فيصل `GET` و `POST` وغيرهما إلى الدالة نفسها.

`@Head` يطلب الرؤوس من غير جسم في عقد HTTP. `@Options` يعلن الخيارات المتاحة للمسار. `@QueryMethod` يسجل الطريقة `QUERY` للعملاء الذين يستخدمونها. الثلاثة موجودون عندما يطلب العقد ذلك، والمثال اليومي يبقى على الخمسة الأوائل.

## مثال عملي

فحص جاهزية يقبل أي طريقة على مسار قصير:

```ts
import { All, Controller, Head } from '@nestjs/common';
import { EditionsService } from './editions.service.js';

@Controller('editions')
export class EditionProbesController {
  constructor(private readonly editions: EditionsService) {}

  @Head('current')
  probe() {
    return this.editions.currentExists();
  }

  @All('ping')
  ping() {
    return { ok: true };
  }
}
```

`HEAD /editions/current` منفصل عن `GET`. `ALL /editions/ping` يجيب أي طريقة على هذا المسار فقط.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `@Get('create')` لإنشاء طبعة | الكتابة تبدو قراءة، والوسيط قد يخزنها |
| `@Put` يعدّل حقلًا واحدًا ويبقي الباقي من نسخة قديمة | العميل يتوقع استبدالًا كاملًا |
| خلط `@QueryMethod` مع `@Query` | الأول طريقة HTTP، والثاني يقرأ `?` |
| `@All()` على جذر المورد | كل فعل، بما فيه الحذف، يدخل الدالة نفسها |

## متى نستخدمه؟

`@Get` للقراءة، `@Post` للإنشاء، `@Put` لاستبدال التمثيل، `@Patch` لتعديل جزئي، `@Delete` للإزالة. `@Head` عندما يكفي وجود المورد. `@All` لمسار تشغيلي ضيق مثل `ping`.

## متى لا نستخدمه؟

لتحديد رمز 204 أو تحويل 301. ذلك `@HttpCode` و `@Redirect`. الطريقة هنا هي الفعل، والرمز درس Status Codes.

## مقارنة

| Decorator | فعل HTTP على المورد |
| --- | --- |
| `@Get` | قراءة |
| `@Post` | إنشاء أو عملية غير متطابقة مع استبدال كامل |
| `@Put` | استبدال التمثيل |
| `@Patch` | تعديل جزئي |
| `@Delete` | حذف |

## تمارين

### تمرين سهل

أي Decorator لـ `POST /editions`؟

<details><summary>الحل</summary>

`@Post()` على Controller بادئته `editions`.

</details>

### تمرين متوسط

العميل يرسل عنوانًا جديدًا فقط. أي طريقة تختار، وماذا تعني للخدمة؟

<details><summary>الحل</summary>

`@Patch`. الخدمة تحدّث الحقول المرسلة وتبقي بقية التمثيل. `@Put` يعني أن الجسم هو التمثيل الكامل بعد الاستبدال.

</details>

### تمرين متقدم

لماذا سُمّي Decorator الطريقة `QUERY` باسم `@QueryMethod`؟

<details><summary>الحل</summary>

`@Query()` محجوز لمعاملات الاستعلام على الدالة. الاسم `QueryMethod` يسجل طريقة HTTP المسماة `QUERY` من غير أن يختلط بمعامل `@Query`.

</details>

## أسئلة مراجعة

- ما الطرق الخمسة التي تغطي CRUD المعتاد؟
- ماذا يسجل `@All()`؟
- أي طريقة نجاحها الافتراضي 201؟

## الخلاصة

Decorator الطريقة هو فعل نقطة النهاية. المسار يسمي المورد، والطريقة تسمي العمل، والخمسة الأساسية تغطي القراءة والإنشاء والاستبدال والتعديل والحذف.

## ماذا بعد؟

التقاط جزء متغير من المسار: Route Params.

## المصادر الرسمية

- NestJS — Controllers (Resources): https://docs.nestjs.com/controllers#resources
