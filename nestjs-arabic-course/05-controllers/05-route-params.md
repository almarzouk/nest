# Route Params

## الهدف من هذا الدرس

التقاط قيمة متغيرة داخل مسار URL عبر رمز مثل `:tracking` وقراءتها بـ `@Param`.

## المشكلة التي يحلها

مسار ثابت `/shipments/item` لا يميز شحنة عن أخرى. المعرف جزء من هوية المورد في REST، فيعيش في المسار، و `@Param` يخرجه للمعالج.

## الفكرة ببساطة

`:tracking` رمز في نص `@Get` أو غيره. `@Param('tracking')` يعيد ذلك الجزء كنص. `@Param()` بلا اسم يعيد الكائن كله.

## التعريف التقني

المسار `GET /shipments/4481` مع `@Get(':tracking')` يملأ الرمز `tracking` بالقيمة `4481`. الاستعلام `?expand=route` يبقى خارج هذا الدرس.

```ts
@Get(':tracking')
findOne(@Param() params: { tracking: string }) {
  return params.tracking;
}

@Get(':tracking')
findOne(@Param('tracking') tracking: string) {
  return tracking;
}
```

القيمة نص. تعليق TypeScript `number` لا يحوّل `"4481"` إلى عدد. التحويل عمل Pipe، وفصله لاحق. من غير Pipe يبقى النص كما التقطه المسار.

أكثر من رمز: `@Get(':carrier/:tracking')` ينتج `params.carrier` و `params.tracking`.

ضع المسارات الثابتة قبل هذا النمط ما دامت الاستراتيجية `declaration` على Express، كما في درس Routing. `@Get('delayed')` بعد `@Get(':tracking')` يجعل كلمة `delayed` تبدو كرقم تتبع.

## Mental Model

```text
GET /shipments/dhl/4481
  @Controller('shipments')
  @Get(':carrier/:tracking')
  @Param('carrier')    -> "dhl"
  @Param('tracking')   -> "4481"
```

## أبسط مثال

```ts
import { Controller, Get, Param } from '@nestjs/common';
import { ShipmentsService } from './shipments.service.js';

@Controller('shipments')
export class ShipmentsController {
  constructor(private readonly shipments: ShipmentsService) {}

  @Get(':tracking')
  findOne(@Param('tracking') tracking: string) {
    return this.shipments.findOne(tracking);
  }
}
```

## شرح المثال

- `Param` مستورد من `@nestjs/common`.
- وسيطة `'tracking'` تطابق اسم الرمز في المسار، حرفًا بحرف.
- الخدمة تستقبل نصًا. إن احتاجت عددًا، التحويل موثّق عندها أو عبر Pipe لاحقًا.
- الدالة تعيد كائن الشحنة، وNest يرسله JSON برمز 200.

## ماذا يحدث داخليًا؟

بعد اختيار المعالج تنسخ Nest `req.params` إلى وسيط `@Param`. مع الاسم تأخذ `req.params.tracking`. من غير اسم تمرر الكائن. هذا بعد مطابقة Routing وقبل جسم الدالة. Pipes المرتبطة بالوسيط، عندما تُضاف لاحقًا، تعمل في مرحلة Pipes وترى هذه القيمة.

اسم في `@Param('code')` مع رمز `:tracking` يعطي `undefined` لأن المفتاح غير موجود على `req.params`.

## مثال عملي

```ts
import { Controller, Get, Param } from '@nestjs/common';
import { ShipmentsService } from './shipments.service.js';

@Controller('shipments')
export class CarrierShipmentsController {
  constructor(private readonly shipments: ShipmentsService) {}

  @Get('delayed')
  delayed() {
    return this.shipments.findDelayed();
  }

  @Get(':carrier/:tracking')
  findOne(
    @Param('carrier') carrier: string,
    @Param('tracking') tracking: string,
  ) {
    return this.shipments.findByCarrier(carrier, tracking);
  }
}
```

`GET /shipments/delayed` يثبت على الدالة الأولى. `GET /shipments/dhl/4481` يملأ الناقل والتتبع. ترتيب الإعلان هنا يحمي المسار الثابت على Express الافتراضي.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| معاملة `@Param('tracking')` كعدد لأن النوع `number` | المقارنة `=== 4481` تفشل والنص `"4481"` هو المخزَّن |
| `@Param('id')` بينما الرمز `:tracking` | القيمة `undefined` والخدمة تبحث بمفتاح فارغ |
| مسار ثابت بعد `@Get(':tracking')` | الكلمة الثابتة تُلتقط كمعامل |
| وضع معرف الشحنة في الاستعلام فقط | عنوان المورد لا يعود كافيًا لإعادة الطلب نفسه |

## متى نستخدمه؟

عندما تميز القيمة موردًا داخل المجموعة: تتبع، رقم طبعة، معرف عضو في مسار مثل `/members/:id`.

## متى لا نستخدمه؟

للترقيم والفرز `?page=2&sort=eta`. ذلك `@Query`. المعامل في المسار جزء من الهوية، والاستعلام يضيّق نتيجة مسار ثابت.

## مقارنة

| الشكل | ماذا يصل إلى الدالة |
| --- | --- |
| `@Param('tracking')` | نص الجزء المسمّى |
| `@Param()` | كائن كل رموز المسار |
| `@Query('tracking')` | قيمة `?tracking=` بعد أن تطابق المسار |

## تمارين

### تمرين سهل

ماذا يعيد `@Param('tracking')` لطلب `GET /shipments/4481`؟

<details><summary>الحل</summary>

النص `"4481"`.

</details>

### تمرين متوسط

المسار `@Get(':carrier/:tracking')`. كيف تقرأ الناقل فقط؟

<details><summary>الحل</summary>

`@Param('carrier') carrier: string`. الكائن الكامل عبر `@Param()` ثم `params.carrier` يؤدي الغرض نفسه.

</details>

### تمرين متقدم

لماذا `@Get(':tracking')` ثم `@Get('delayed')` يجعل قائمة المتأخرين فارغة أو خاطئة على Express الافتراضي؟

<details><summary>الحل</summary>

`delayed` يطابق `:tracking` أولًا، فتعمل `findOne("delayed")` وتبحث عن شحنة بهذا المعرف. المسار الثابت يُكتب قبل نمط المعامل، أو تُفعَّل استراتيجية `specificity` كما في درس Routing.

</details>

## أسئلة مراجعة

- أين يُعلَن رمز Route Param؟
- ما نوع القيمة القادمة من `@Param` قبل أي Pipe؟
- متى تختار `@Param()` بلا اسم؟

## الخلاصة

`:name` يلتقط قطعة من المسار و `@Param` يمررها كنص. المسارات الثابتة تُحمى بترتيب الإعلان، والتحويل إلى عدد يبقى عمل Pipe لاحق.

## ماذا بعد؟

قيم `?` بعد المسار: Query Params.

## المصادر الرسمية

- NestJS — Route parameters: https://docs.nestjs.com/controllers#route-parameters
