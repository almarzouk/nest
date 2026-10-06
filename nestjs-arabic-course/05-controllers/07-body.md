# Body

## الهدف من هذا الدرس

قراءة جسم الطلب بـ `@Body`: الكائن كاملًا أو حقلًا واحدًا، ووصف الشكل بصنف يبقى وقت التشغيل.

## المشكلة التي يحلها

إنشاء إعارة يحتاج ISBN ومعرف العضو في JSON، لا في المسار. مسار `POST /loans` ثابت، والحمولة في الجسم. `@Body` هو المدخل الذي يمرر هذه الحمولة إلى المعالج.

## الفكرة ببساطة

بعد محلّل JSON يصبح `req.body` كائنًا. `@Body()` يمرره كاملًا. `@Body('isbn')` يمرر حقلًا. صنف DTO يصف الشكل ويبقى موجودًا بعد الترجمة، والواجهة TypeScript تُمسح.

## التعريف التقني

DTO كائن يصف شكل البيانات على الشبكة. التوثيق يفضّل صنفًا لأن الصنف كيان JavaScript. الواجهة تُحذف عند الترجمة، فلا تجد Nest نوعًا وقت التشغيل عندما تحتاج Pipes إلى metatype.

```ts
export class CreateLoanDto {
  isbn: string;
  memberId: string;
}
```

الصنف هنا وصف. `ValidationPipe` الذي يحذف الحقول الزائدة ويتحقق من الأنواع يأتي في فصل Validation. من غير Pipe تصل الحقول كما فكّها المحلّل، بما فيها مفاتيح لم تُذكر في الصنف.

`@Body()` و `@Body('isbn')` يقبلان لاحقًا كائن خيارات فيه `schema` و `pipes` لربط Standard Schema. ربط المخطط Metadata فقط. التحقق يتم بعد تسجيل `StandardSchemaValidationPipe` أو Pipe يقرأ `metadata.schema`. ذلك فصل لاحق؛ هنا نقرأ الجسم.

JSON غير الصالح يتوقف عند المحلّل ولا تدخل الدالة. مع المحلّل المعتاد، جسم غائب على طلب JSON يعطي كائنًا فارغًا.

## Mental Model

```text
POST /loans
Content-Type: application/json
{"isbn":"9780001","memberId":"m-9"}

@Post()
create(@Body() dto: CreateLoanDto)
  dto.isbn     -> "9780001"
  dto.memberId -> "m-9"
```

## أبسط مثال

```ts
import { Body, Controller, Post } from '@nestjs/common';
import { LoansService } from './loans.service.js';
import { CreateLoanDto } from './create-loan.dto.js';

@Controller('loans')
export class LoansController {
  constructor(private readonly loans: LoansService) {}

  @Post()
  create(@Body() dto: CreateLoanDto) {
    return this.loans.open(dto);
  }
}
```

## شرح المثال

- `@Post()` على بادئة `loans` يستقبل الإنشاء. الرمز الافتراضي 201 لأن الطريقة POST.
- `CreateLoanDto` نوع الوسيط. وقت التشغيل القيمة هي كائن الجسم، والصنف يفيد لاحقًا أي Pipe تعتمد على metatype.
- `open` في الخدمة تقرر هل الإعارة مسموحة. Controller لا يفحص المخزون.
- إعادة نتيجة `open` تجعل Nest ترسل JSON.

## ماذا يحدث داخليًا؟

Middleware المحلّل، قبل اختيار تفاصيل المعالج التي تهمك هنا، يملأ `req.body`. `@Body()` ينسخ ذلك الكائن إلى الوسيط. `@Body('isbn')` يأخذ الخاصية. ترتيب القراءة داخل الدالة هو ترتيب الوسائط التي كتبتها.

حقل غائب يكون `undefined` على الكائن. حقل زائد مثل `note` يبقى موجودًا إلى أن يزيله Pipe. الواجهة التالية لا تغيّر هذا السلوك لأنها تُمسح:

```ts
interface CreateLoanDto {
  isbn: string;
}
```

استخدم الصنف عندما تتوقع Pipe لاحقًا. للتجربة القصيرة قبل فصل التحقق، الصنف ما زال أوضح عقدًا للمبرمج التالي.

## مثال عملي

تجديد إعارة يقرأ الجسم كاملًا، ومسار ضيق يقرأ الأيام فقط:

```ts
import { Body, Controller, Param, Patch, Post } from '@nestjs/common';
import { LoansService } from './loans.service.js';
import { RenewLoanDto } from './renew-loan.dto.js';

@Controller('loans')
export class LoanRenewalsController {
  constructor(private readonly loans: LoansService) {}

  @Patch(':id')
  renew(@Param('id') id: string, @Body() dto: RenewLoanDto) {
    return this.loans.renew(id, dto);
  }

  @Post('estimate')
  estimate(@Body('days') days: number) {
    return this.loans.estimateFee(days);
  }
}
```

`days` في JSON رقمًا يصل رقمًا لأن JSON يميز الأعداد. هذا غير `@Query` الذي يصل نصًا. تعليق النوع `number` يوثّق العقد، والمحلّل هو من أنشأ العدد إن كان الجسم `{"days":14}`. نص `"14"` داخل JSON يبقى نصًا.

`RenewLoanDto` صنف بحقل `days`. مسار `:id` يحدد الإعارة، والجسم يحدد التعديل.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| وصف الجسم بواجهة فقط ثم توقع أن Pipe لاحقًا ترى النوع | الواجهة تُمسح و metatype فارغ |
| اعتبار الصنف تحققًا | الحقول الزائدة والناقصة تصل حتى يُسجَّل Pipe |
| قراءة `@Body('days')` وتوقع تحويل `"14"` النصي | JSON يحافظ على نوع القيمة المكتوبة |
| وضع ISBN في الاستعلام على `POST` | الحمولة التي تُعاد وتُسجَّل تنتمي إلى الجسم |

## متى نستخدمه؟

على `@Post` و `@Put` و `@Patch` عندما يحمل العميل تمثيلًا أو تعديلًا. `@Body()` للعقد الكامل، و `@Body('field')` لحقل واحد في عملية ضيقة.

## متى لا نستخدمه؟

على `GET` الذي لا جسم له في الاستعمال المعتاد. التصفية `@Query`، والهوية `@Param`. التحقق من إلزام الحقول فصل Validation.

## مقارنة

| الوسيلة | ماذا تصل إليه |
| --- | --- |
| `@Body()` | كائن `req.body` |
| `@Body('isbn')` | خاصية واحدة أو `undefined` |
| صنف DTO | شكل موثّق يبقى وقت التشغيل لـ Pipes لاحقة |
| واجهة TypeScript | فحص المترجم فقط، ثم تُحذف |

## تمارين

### تمرين سهل

أي Decorator يقرأ `{"isbn":"9780001"}` على `POST /loans`؟

<details><summary>الحل</summary>

`@Body()` على وسيط الدالة، أو `@Body('isbn')` إن لزم الحقل فقط.

</details>

### تمرين متوسط

لماذا يفضّل التوثيق صنف `CreateLoanDto` على واجهة بالشكل نفسه؟

<details><summary>الحل</summary>

الصنف يبقى في JavaScript بعد الترجمة، فتستطيع Pipes قراءة metatype. الواجهة تُحذف ولا تترك رمزًا وقت التشغيل.

</details>

### تمرين متقدم

الجسم `{"isbn":"9780001","note":"هدية"}` والصنف يذكر `isbn` فقط. ماذا يصل إلى `open` قبل أي Pipe؟

<details><summary>الحل</summary>

الكائن كاملًا بما فيه `note`. الصنف لا يحذف المفاتيح. الحذف سلوك `ValidationPipe` أو Pipe مكافئ في فصل لاحق.

</details>

## أسئلة مراجعة

- ماذا يقرأ `@Body()` بعد محلّل JSON؟
- لماذا لا تكفي الواجهة عندما ستُضاف Pipes؟
- كيف تقرأ حقلًا واحدًا باسم `days`؟

## الخلاصة

`@Body` يمرر جسم الطلب إلى المعالج. صنف DTO يوثّق الشكل ويبقى للخطوات اللاحقة، والتحقق نفسه Pipe يأتي بعد هذا الفصل.

## ماذا بعد؟

رؤوس الطلب الواردة: Headers.

## المصادر الرسمية

- NestJS — Request payloads: https://docs.nestjs.com/controllers#request-payloads
