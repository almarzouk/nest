# مرحلة Guards

## الهدف من هذا الدرس

تثبيت ترتيب Guards: العام، ثم Controller، ثم المسار، وداخل المستوى ترتيب الربط.

## المشكلة التي يحلها

حارس «المكتبة مفتوحة» وحارس «العضو أمين» وحارس «هذا `PATCH` مسموح» تتداخل إن لم يُعرف من يعمل أولًا. تشغيل حارس المسار قبل العام يترك قاعدة عامة ناقصة على بعض الدوال. Nest تمشي من الخارج إلى الدالة.

## الفكرة ببساطة

Guard يقرر هل يكمل الطلب نحو Interceptors و Pipes والمعالج. العام عبر `app.useGlobalGuards()` أو الرمز `APP_GUARD` يسبق الزينة على الصنف، وتلك تسبق الزينة على الدالة. `@UseGuards(Guard1, Guard2)` يشغّل `Guard1` ثم `Guard2`.

تنفيذ `canActivate` فصل Guards. هنا الترتيب وأثر التوقف.

## التعريف التقني

من التوثيق، على Controller:

```ts
@UseGuards(Guard1, Guard2)
@Controller('vaults')
export class VaultsController {
  @UseGuards(Guard3)
  @Get()
  list() {
    return this.vaults.list();
  }
}
```

`Guard1` قبل `Guard2`، وكلاهما قبل `Guard3`. أي حارس عام يسبق `Guard1`.

«عام» يعني الربط بـ `app.useGlobalGuards()` أو بمزوّد `APP_GUARD`. «Controller» زينة قبل الصنف. «مسار» زينة قبل الدالة. داخل المستوى الواحد الترتيب هو ترتيب الربط.

إعادة `false` تمنع الطلب: Nest ترمي استثناء رفض، فتنتقل الدورة إلى Exception Filters ولا تبدأ Pipes ولا المعالج. الرمي من الحارس يقفز بالطريقة نفسها. النجاح يستدعي الحارس التالي ثم مرحلة Interceptors الدخول.

## Mental Model

```text
حراس APP_GUARD بترتيب التسجيل
  -> Guard1
  -> Guard2
  -> Guard3
  -> Interceptors قبل المعالج
```

توقف أي حارس يقطع هذا السهم.

## أبسط مثال

```ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { Guard1 } from './guard1.js';
import { Guard2 } from './guard2.js';
import { Guard3 } from './guard3.js';
import { VaultsService } from './vaults.service.js';

@UseGuards(Guard1, Guard2)
@Controller('vaults')
export class VaultsController {
  constructor(private readonly vaults: VaultsService) {}

  @UseGuards(Guard3)
  @Get()
  list() {
    return this.vaults.list();
  }
}
```

## شرح المثال

- `list` لا تُستدعى إلا بعد نجاح الثلاثة وبعد Interceptors و Pipes التي تليهم.
- `Guard3` لا يعوّض عن فشل `Guard1`. الأول لم يمرر التحكم.
- الخدمة مكان القاعدة. الحارس لا يحسب غرامة ولا يفتح خزنة.
- الأسماء `Guard1` كما في التوثيق لبيان الترتيب، وبناء الصنف في فصل Guards.

## ماذا يحدث داخليًا؟

بعد Middleware يكون المسار قد حُدد، لذلك زينة الصنف والدالة معروفتان. Nest تجمع الحراس من العام إلى الدالة وتناديهم بالترتيب. `true` يكمل. `false` يصبح منعًا والاستثناء يدخل المرشحات من مرشح المسار إلى Controller ثم العام. هذا غير استثناء Middleware الذي لا يراه إلا العام.

عدة مزودات `APP_GUARD` تعمل بترتيب التسجيل قبل حراس الصنف، وما زالت داخل البند «العام». Scope `REQUEST` على حارس يغيّر عمر النسخة ولا يغيّر موضع المرحلة.

## مثال عملي

```text
APP_GUARD LibraryOpenGuard
@UseGuards(MemberGuard) على VaultsController
@UseGuards(PatchGuard) على PATCH :id
```

`PATCH /vaults/9` يسأل: هل المكتبة مفتوحة، ثم هل العضو معروف، ثم هل التعديل مسموح. `GET /vaults` يسأل الأولين فقط. فشل `MemberGuard` لا يشغّل `PatchGuard` ولا Pipe على `:id`.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| وضع الحارس العام على دالة واحدة فقط | بقية المسارات بلا القاعدة العامة |
| توقع `Guard3` قبل `Guard1` | `@UseGuards` من اليسار إلى اليمين بعد العام |
| فحص الصلاحية داخل Service بعد Pipe ثقيل | التحقق من الجسم تم لطالب قد يُرفض |
| اعتبار فشل Guard استثناء Middleware | المسار مختار، فمرشحات المسار و Controller مشاركة |

## متى نستخدمه؟

عندما يكون السؤال «هل يُسمح لهذا الطلب بتجاوز هذا المسار؟» بعد معرفة المعالج وقبل تحويل الجسم وتنفيذ القاعدة.

## متى لا نستخدمه؟

لختم كل الطلبات قبل اختيار المسار. ذلك Middleware. لتحويل `id` أو شكل JSON. ذلك Pipe. لحساب نتيجة الإعارة. ذلك Service بعد نجاح السلسلة.

## مقارنة

| مكان الربط | موقعه في المرحلة |
| --- | --- |
| `app.useGlobalGuards` أو `APP_GUARD` | أول الحراس |
| `@UseGuards` على الصنف | بعد العام وقبل الدالة |
| `@UseGuards(Guard1, Guard2)` | `Guard1` ثم `Guard2` |
| `@UseGuards` على الدالة | آخر الحراس |

## تمارين

### تمرين سهل

من يعمل أولًا: `Guard3` على `list` أم `Guard1` على الصنف؟

<details><summary>الحل</summary>

`Guard1`. حراس الصنف تسبق حراس الدالة، والعام يسبق الاثنين.

</details>

### تمرين متوسط

`MemberGuard` أعاد `false`. هل تعمل `ParseIntPipe` على `:id`؟

<details><summary>الحل</summary>

لا. المنع ينهي المرحلة باستثناء رفض، و Pipes لا تبدأ.

</details>

### تمرين متقدم

لماذا يرى مرشح على `PATCH` فشل Guard بينما لا يرى استثناء `app.use`؟

<details><summary>الحل</summary>

Guard يعمل بعد اختيار المسار، فمرشحات المسار ثم Controller ثم العام قادرة على الرؤية، وأول من يمسك يوقف الباقي. استثناء Middleware قبل الاختيار، فلا ينطبق إلا المرشح العام.

</details>

## أسئلة مراجعة

- ما ترتيب المستويات الثلاثة؟
- ماذا يحدث عند `false`؟
- أين يقف `APP_GUARD` بالنسبة لزينة الدالة؟

## الخلاصة

Guards من العام إلى Controller إلى المسار، وبالترتيب المكتوب داخل `@UseGuards`. النجاح يكمل نحو Interceptors. الفشل يمنع Pipes والمعالج وينقل الاستثناء إلى مرشحات مسار معروف.

## ماذا بعد؟

الدخول إلى اللف: Interceptors قبل Controller.

## المصادر الرسمية

- NestJS — Request lifecycle (Guards): https://docs.nestjs.com/faq/request-lifecycle
