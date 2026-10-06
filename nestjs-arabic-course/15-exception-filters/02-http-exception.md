# HttpException

## الهدف من هذا الدرس

رمي `HttpException` بجسم ورمز واضحين، مع سبب داخلي لا يُرسل إلى العميل.

## المشكلة التي يحلها

`Error` العادي يصبح `500` ويخفي أن الطلب ممنوع أو أن التذكرة غير موجودة. العميل لا يستطيع أن يفرّق عطل الخادم عن رفض مفهوم.

## الفكرة ببساطة

`new HttpException(response, status)` يبني استثناء تعرفه الطبقة المدمجة. `response` نص أو كائن. `status` رمز HTTP من تعداد `HttpStatus`. وسيط ثالث اختياري `options.cause` للسجل لا للجسم.

## التعريف التقني

الصنف من `@nestjs/common`. المثال الأساسي في الصفحة:

```ts
throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
```

الجسم الناتج:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

النص في الوسيط الأول يستبدل `message`. الكائن يستبدل الجسم كله كما تسلسله Nest. الصفحة تضرب مثالًا:

```ts
throw new HttpException(
  { status: HttpStatus.FORBIDDEN, error: 'Desk closed' },
  HttpStatus.FORBIDDEN,
  { cause: error },
);
```

فيصبح الرد:

```json
{
  "status": 403,
  "error": "Desk closed"
}
```

لاحظ أن مفاتيح الكائن هي ما يخرج. `statusCode` الافتراضي لا يُضاف تلقائيًا حين تمرر كائنًا. اختر شكلًا واحدًا للمشروع.

`cause` لا يُسلسَل إلى العميل. يفيد من يسجّل الاستثناء الأصلي، مثل فشل قاعدة ترجمته إلى رمز مفهوم.

`getStatus()` و `getResponse()` على النسخة للفلتر المخصص لاحقًا.

الرمز من `HttpStatus` لا من رقم سحري، كما تطلب الصفحة. `HttpStatus.NOT_FOUND` أوضح من `404` في المراجعة، والرقم نفسه على السلك.

هذا الصنف وفئاته `IntrinsicException`. الفلتر المدمج لا يطبعها كعطل غير متوقع. إن أردت سجلًا لكل `404` فاكتبه في فلتر أو مسجّل مقصود، لا تنتظر الطرفية الافتراضية.

## Mental Model

```text
نص + رمز     → { statusCode, message }
كائن + رمز   → الكائن كما هو
cause        → للتشخيص، ليس في JSON
```

## أبسط مثال

```ts
import { HttpException, HttpStatus } from '@nestjs/common';

export function missingTicket(id: string): HttpException {
  return new HttpException(`Ticket ${id} was not found`, HttpStatus.NOT_FOUND);
}
```

```ts
@Get(':id')
find(@Param('id', ParseIntPipe) id: number): never {
  throw missingTicket(String(id));
}
```

العميل يرى `404` والنص الذي كتبته. `ParseIntPipe` إن فشل يرمي `BadRequestException` قبل الدالة، وهو فرع آخر.

## شرح المثال

- الدالة المساعدة توحّد النص حتى لا يكتب كل مسار جملة مختلفة.
- `HttpStatus.NOT_FOUND` هو `404`.
- لا `try/catch` حول الرمي في المتحكم. التصعيد مقصود.
- `id` في النص مقبول لتذكرة. لا تضع استعلام SQL ولا مسار ملف في `message`.

## ماذا يحدث داخليًا؟

الرمي يخرج من الدالة إلى منطقة الاستثناءات. الفلتر المدمج يرى `HttpException`، يقرأ الحالة من الوسيط الثاني، ويبني الجسم من النص أو من الكائن. لا يقرأ `cause`. الاتصال يُغلق بجسم JSON ورمز الحالة.

إن كان على المسار فلتر `@Catch(HttpException)` فذلك الفلتر أولى من العام، حسب ترتيب الدرس الأخير. من غير فلتر مخصص يكفي المدمج.

رمي `HttpException` من الحارس أو من الأنبوب يستخدم الطريق نفسه لأن المسار مختار. من Middleware يصل إلى الفلتر العام المدمج أو إلى فلتر عام سجّلته، لا إلى فلتر الدالة.

## مثال عملي

`TicketsService` لا تجد الصف فترمي `missingTicket`. الاختبار يتوقع `404` و `message` فيها المعرف. اختبار آخر يمرر `cause: queryError` ويتوقع أن الجسم JSON بلا المكدس وبلا اسم الجدول. السجل في الاختبار يقرأ `exception.cause` إن كتبت فلترًا لاحقًا. حتى ذلك الحين `cause` جاهز ولا يغيّر العميل.

## أخطاء شائعة

- تمرير كائن فيه `stack`.
- رمز `200` داخل `HttpException`. الاستثناء للخطأ لا للنجاح.
- التقاطه في المتحكم وإرجاع `{ message }` مع رمز نجاح افتراضي `200` أو `201`.
- رقم حالة من سلسلة قادمة من العميل.

## متى نستخدمه؟

حين لا توجد فئة جاهزة تطابق المعنى، أو حين تحتاج جسمًا بمفاتيح خاصة بكم مع رمز صحيح.

## متى لا نستخدمه؟

لغياب تذكرة إن كانت `NotFoundException` تكفي بالرسالة الافتراضية أو برسالة نصية. الفئات الجاهزة الدرس التالي. ولا لنجاح بطيء. النجاح `return`.

## مقارنة

| البناء | الجسم |
| --- | --- |
| `HttpException('Forbidden', 403)` | `statusCode` و `message` |
| `HttpException({ error }, 403)` | المفاتيح التي وضعتها |
| `Error('Forbidden')` | `500` عام |
| `cause` | غير ظاهر للعميل |

## تمارين

### تمرين سهل

ما الوسيطان المطلوبان لبناء `HttpException`؟

<details>
<summary>الحل</summary>

جسم الرد، نص أو كائن، ورمز الحالة. `cause` اختياري في وسيط ثالث.

</details>

### تمرين متوسط

لماذا قد يختفي `statusCode` من JSON حين تمرر كائنًا؟

<details>
<summary>الحل</summary>

الكائن يستبدل الجسم كله. إن لم تضع `statusCode` داخل الكائن فلن يُضاف تلقائيًا كما في مسار النص.

</details>

### تمرين متقدم

فشل استعلام يجب أن يصبح `404` للعميل مع بقاء الخطأ الأصلي للسجل. ماذا تضع في البناء، وما الذي تتأكد أنه ليس في الرد؟

<details>
<summary>الحل</summary>

`new HttpException('Ticket was not found', HttpStatus.NOT_FOUND, { cause: error })`. الرد بلا `cause` وبلا `stack`. السجل يقرأ `cause` من الاستثناء في الفلتر أو في مكان التسجيل.

</details>

## أسئلة مراجعة

- كيف يغيّر النص الجسم الافتراضي؟
- أين يعيش `cause`؟
- لماذا لا يُطبع الاستثناء كعطل غير متوقع في الفلتر المدمج؟
- ماذا يرى العميل عند `Error` العادي بدل هذا الصنف؟

## الخلاصة

`HttpException` يربط رمزًا بجسم نصي أو كائن. `cause` تشخيص داخلي. العميل يرى ما وضعته في الوسيط الأول، لا مكدس الخطأ الأصلي.

## ماذا بعد؟

الدرس التالي فئات جاهزة: `BadRequestException` و `UnauthorizedException` و `ForbiddenException` و `NotFoundException` و `ConflictException`.

## المصادر الرسمية

- [Exception filters](https://docs.nestjs.com/exception-filters)
