# مقدمة ValidationPipe

## الهدف من هذا الدرس

تعريف `ValidationPipe` المدمج: يفحص معاملًا نوعه صنف عليه زينة `class-validator`، ويرمي `400` عند الفشل.

## المشكلة التي يحلها

`ParseIntPipe` يفحص قيمة واحدة. جسم إنشاء فاتورة فيه عدة حقول. فحصها يدويًا في المتحكم يكرر القواعد ويبقيها بعيدة عن شكل البيانات.

## الفكرة ببساطة

تكتب صنفًا بحقول مزيّنة. المعامل `@Body() invoice: CreateInvoiceDto`. أنبوب `ValidationPipe` يقرأ الزينة ويرفض الجسم الذي يخالفها. خيارات مثل `whitelist` فصل لاحق. هنا أصغر تشغيل.

## التعريف التقني

الأنبوب مُصدَّر من `@nestjs/common`. يحتاج الحزمتين:

```bash
npm i class-validator class-transformer
```

يربط عالميًا:

```ts
app.useGlobalPipes(new ValidationPipe());
```

أو على معامل: `@Body(new ValidationPipe())`.

الصنف يجب أن يكون صنفًا حقيقيًا في وقت التشغيل. الواجهة والنوع العام لا يصدران بيانات وصفية، والاستيراد `import type` يُمسح. اكتب `import { CreateInvoiceDto }`.

عند فشل `@IsEmail()` مثلًا يكون الرد `400` و `message` مصفوفة رسائل. شكل الرسائل وخيارات `errorFormat` درسهما في فصل التحقق.

الأنبوب يتخطى الأنواع الأصلية `String` و `Boolean` و `Number` و `Array` و `Object` لأن الزينة لا تُعلَّق عليها. لذلك `@Param('id') id: string` لا يُفحص بهذا الأنبوب. استخدم `ParseIntPipe` لذلك المعامل.

## Mental Model

```text
JSON خام → صنف CreateInvoiceDto → قراءة الزينة → نجاح: القيمة تمر
                                              → فشل: 400 والمعالج لا يعمل
```

العقد وقت التشغيل هو الصنف، لا واجهة TypeScript.

## أبسط مثال

```ts
import { IsInt, IsString, MinLength } from 'class-validator';

export class CreateInvoiceDto {
  @IsString()
  @MinLength(3)
  customer: string;

  @IsInt()
  cents: number;
}
```

```ts
import { Body, Controller, Post } from '@nestjs/common';
import { CreateInvoiceDto } from './create-invoice.dto.js';

@Controller('invoices')
export class InvoicesController {
  @Post()
  create(@Body() dto: CreateInvoiceDto): CreateInvoiceDto {
    return dto;
  }
}
```

مع الأنبوب العام، الجسم `{ "customer": "ن", "cents": 100 }` يُرفض لأن `customer` أقصر من 3. الجسم الصالح يصل إلى `create`.

## شرح المثال

- الزينة من `class-validator` لا من Nest. الأنبوب يستدعي المكتبة.
- `@Body()` بلا أنبوب محلي يكفي حين يكون `ValidationPipe` عامًا.
- الدالة ترجع الـ DTO في هذا المثال حتى ترى ماذا مر. الخدمة الحقيقية تُستدعى بعد النجاح فقط.
- `cents` كعدد في JSON يعمل مع `@IsInt()`. الاستعلام النصي ليس هذا المسار.

لا تضبط `transform: true` هنا. تحويل الكائن العادي إلى نسخة الصنف خيار لاحق، والتحقق الافتراضي يستطيع قراءة الزينة عبر `plainToInstance` داخليًا كما تشرح صفحة الأنابيب في العينة التعليمية. السلوك الذي نثبّته الآن: الفشل `400` والنجاح يمرر القيمة إلى الدالة.

## ماذا يحدث داخليًا؟

1. المعالج مُعلَّم بأن نوع الجسم `CreateInvoiceDto`.
2. `emitDecoratorMetadata` مع `reflect-metadata` يضع `design:type` وبيانات الزينة على الصنف. مشاريع Nest CLI تفعّل ذلك.
3. الأنبوب يرى `metatype` صنفًا ليس من الأنواع الأصلية.
4. يستدعي التحقق. مصفوفة أخطاء غير فارغة تصبح `BadRequestException`.
5. الدالة لا تُستدعى عند الفشل.

حقل زائد مثل `role` يمر في هذا الإعداد لأن `whitelist` مطفأ. لا تعتمد على هذا الدرس لمنع الحقول الزائدة.

## مثال عملي

عميل يرسل `{ "customer": "دار النشر", "cents": 2500 }`. الأنبوب ينجح. `InvoicesService.create` تستلم الكائن. عميل آخر يرسل `cents` كنص `"2500"` من نموذج خاطئ. `@IsInt()` يرفض قبل أن تجمع الخدمة قروشًا. رسالة `400` تكفي لإصلاح النموذج. تفاصيل شكل الرسالة تُضبط لاحقًا.

## أخطاء شائعة

- تعريف العقد كـ `interface` ثم الدهشة أن كل جسم يمر. الأنبوب لا يرى الواجهة.
- `import type { CreateInvoiceDto }` في ملف المتحكم. الصنف يختفي وقت التشغيل.
- توقع فحص `@Param('id') id: string` بهذا الأنبوب. النوع الأصلي يُتخطى.
- جمع كل خيارات الإنتاج في أول ربط. المقدمة تثبت الربط والرفض فقط.

## متى نستخدمه؟

على جسم أو كائن استعلام أو معاملات مسار ممثلة بصنف فيه زينة `class-validator`، غالبًا ربطًا عامًا واحدًا.

## متى لا نستخدمه؟

لعدد في مسار واحد حيث `ParseIntPipe` أصرح، ولمخطط Zod حيث الأنبوب الآخر في فصل التحقق. ولا تغلقه على Middleware.

## مقارنة

| | `ParseIntPipe` | `ValidationPipe` |
| --- | --- | --- |
| موضوعه | قيمة واحدة | صنف بحقول |
| القاعدة | مدمجة | زينة `class-validator` |
| الربط الشائع | على المعامل | عام على التطبيق |

## تمارين

### تمرين سهل

من أين تُستورد `ValidationPipe`، وما الحزمتان اللتان تثبّتهما؟

<details>
<summary>الحل</summary>

من `@nestjs/common`. الحزمتان `class-validator` و `class-transformer`.

</details>

### تمرين متوسط

لماذا يمر `@Query('q') q: string` بلا فحص رغم وجود أنبوب عام؟

<details>
<summary>الحل</summary>

`metatype` هو `String`، والأنبوب يتخطى الأنواع الأصلية. ضع القاعدة على صنف، أو استخدم أنبوب تحليل إن كان الشكل عددًا أو منطقًا.

</details>

### تمرين متقدم

زميل كتب `export interface CreateInvoiceDto` ووضع الزينة على حقولها. ماذا يصل إلى `create` عند إرسال `cents` كنص؟

<details>
<summary>الحل</summary>

الواجهة تُمسح ولا تحمل زينة وقت التشغيل. الأنبوب لا يفحص، والنص يصل إلى الدالة. العقد يجب أن يكون `class`.

</details>

## أسئلة مراجعة

- ماذا يحدث للمعالج حين تفشل الزينة؟
- لماذا لا تصلح `interface` كـ DTO لهذا الأنبوب؟
- ما أثر `import type` على الصنف؟
- أي خيار نؤجله إلى فصل التحقق؟

## الخلاصة

`ValidationPipe` يفحص صنفًا مزينًا ويرفض بـ `400` قبل المتحكم. الأنواع الأصلية تُتخطى. الخيارات التي تحذف حقولًا أو تحوّل النسخ أو تغيّر شكل الخطأ تأتي في فصلها، بعد أن يوجد عقد DTO أوضح.

## ماذا بعد؟

الدرس التالي يكتب أنبوبًا خاصًا بنطاق الفواتير يطبّق `PipeTransform` من غير إعادة اختراع الأنابيب المدمجة.

## المصادر الرسمية

- [Pipes](https://docs.nestjs.com/pipes)
- [Validation](https://docs.nestjs.com/application/validation)
