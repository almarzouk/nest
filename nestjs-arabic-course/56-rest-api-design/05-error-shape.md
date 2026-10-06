# شكل الخطأ

## الهدف من هذا الدرس

تثبيت جسم واحد لأخطاء HTTP في رف، يربط الرمز الذي اختارته الاستراتيجية بحقول يقرأها كل عميل، بلا SQL وبلا شكل يختلف بين المسارات.

## المشكلة التي يحلها

التحقق يعيد `message` كمصفوفة، وفلتر ISBN يعيد `error` نصًا، وفلتر 500 يعيد `msg`. الواجهة تفحص ثلاثة أشكال قبل أن تعرض جملة. أحيانًا يصل `stack` في الإنتاج.

## الفكرة ببساطة

كل خطأ على السلك كائن واحد: `statusCode` و `code` ثابت و `message` للإنسان. التفاصيل القابلة للعرض، مثل حقول التحقق، مصفوفة اختيارية. المعرّف الداخلي للسجل ليس سر القاعدة؛ يمكن تمرير `requestId` إن وُجد. لا `stack` ولا استعلام.

## التعريف التقني

شكل Nest الافتراضي لـ `HttpException` غالبًا `{ statusCode, message, error }`. عقد رف يثبّت هذا الشكل أو يضيّقه، لكن لا يتركه يتفرع. `message` قد تكون نصًا. لحقول التحقق، أضف `details: { field, message }[]` بدل تغيير `message` مرة نصًا ومرة مصفوفة إن أردت عميلًا بسيطًا. `code` قيمة مغلقة مثل `book_not_found` يبرمج ضدها العميل، بينما `message` قد تُترجم لاحقًا. الفلتر من فصل الأخطاء هو المكان الذي يكتب هذا الكائن. المتحكم لا يجمع JSON خطأ يدويًا.

## Mental Model

```text
{
  statusCode: 404,
  code: "book_not_found",
  message: "Book not found"
}
```

النجاح لا يستخدم هذا الغلاف. كتاب ناجح يبقى تمثيل كتاب.

## أبسط مثال

```ts
export interface ShelfErrorBody {
  statusCode: number;
  code: string;
  message: string;
  details?: { field: string; message: string }[];
}

export function shelfError(statusCode: number, code: string, message: string): ShelfErrorBody {
  return { statusCode, code, message };
}
```

## شرح المثال

- `code` لا يتغير عندما تُصاغ `message` بالعربية.
- `statusCode` يكرر الرمز داخل الجسم لأن بعض العملاء يقرأون JSON فقط بعد أن ابتلع وسيط الرمز. الرمز على السطر الأول يبقى الأصل.
- لا حقل `debug`.

## ماذا يحدث داخليًا؟

1. الصنف أو الـ Pipe يحدد المعنى.
2. الفلتر يختار `statusCode` و `code` من الجدول.
3. `message` جملة قصيرة بلا أسماء جداول.
4. التحقق يملأ `details` من أخطاء الحقول.
5. العطل غير المتوقع يستخدم `code: "internal_error"` ونفس الجملة الثابتة، والتفاصيل في السجل.

## مثال عملي

```ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpStatus } from '@nestjs/common';
import type { Response } from 'express';
import { BookNotFound } from '../books/books.errors.js';

@Catch(BookNotFound)
export class BookErrorShapeFilter implements ExceptionFilter {
  catch(exception: BookNotFound, host: ArgumentsHost) {
    const body = {
      statusCode: HttpStatus.NOT_FOUND,
      code: 'book_not_found',
      message: 'Book not found',
      bookId: exception.bookId,
    };
    host.switchToHttp().getResponse<Response>().status(404).json(body);
  }
}
```

`bookId` مقبول إن كان العقد يعد به لصاحب الطلب. لا تضف `exception.stack`. تحقق DTO يمكن أن يعيد:

```json
{
  "statusCode": 400,
  "code": "validation_failed",
  "message": "Invalid book",
  "details": [{ "field": "title", "message": "title should not be empty" }]
}
```

## أخطاء شائعة

- `message` مصفوفة في مسار ونص في آخر.
- وضع رسالة Prisma في `message`.
- غلاف خطأ حول `GET /books` الناجح: `{ data, error: null }` مع الإبقاء على أخطاء بلا غلاف. اختر: النجاح تمثيل، والفشل كائن الخطأ، والتزم.
- `code` جملة عربية متغيرة فيتعطل فرع الواجهة.

## متى نستخدمه؟

لكل 4xx و 500 يخرج من رف. وثّقه في OpenAPI كاستجابة مكوّن واحد.

## متى لا نستخدمه؟

لا تلف 201 الناجح بهذا الشكل. ولا تضع أسرار التشخيص في `details` لأنها تذهب إلى جهاز العميل.

## مقارنة

| الحقل | يثبت |
| --- | --- |
| `statusCode` | يطابق سطر HTTP |
| `code` | فرع برمجي مستقر |
| `message` | جملة عرض |
| `details` | حقول التحقق عند الحاجة |

## تمارين

### تمرين سهل

عميل يريد تلوين رسالة ISBN. على أي حقل يتفرع: `message` أم `code`؟

<details>
<summary>الحل</summary>

على `code` مثل `isbn_taken`. `message` للعرض وقد تتغير صياغتها.

</details>

### تمرين متوسط

التحقق يعيد `message: string[]` ومسار المجال يعيد `message: string`. ما التوحيد الأدنى؟

<details>
<summary>الحل</summary>

`message` نص واحد في الحالتين، و `details` مصفوفة للحقول عند التحقق فقط. العميل يقرأ `message` دائمًا كنص.

</details>

### تمرين متقدم

500 من انقطاع القاعدة. اكتب الجسم الذي يجوز أن يخرج، واذكر سطرًا ممنوعًا.

<details>
<summary>الحل</summary>

`{ statusCode: 500, code: "internal_error", message: "Internal server error" }`. الممنوع أي `stack` أو `meta` من Prisma أو سلسلة اتصال داخل `message` أو `details`.

</details>

## أسئلة مراجعة

- أي حقل ثابت للبرمجة وأي حقل للعرض؟
- لماذا لا يختلف شكل 404 عن شكل 400 في المفاتيح الأساسية؟
- أين يُكتب الشكل: في كل متحكم أم في الفلتر؟
- ماذا يغيب عن جسم 500 عمدًا؟

## الخلاصة

خطأ رف كائن بمفاتيح ثابتة: `statusCode` و `code` و `message`، و `details` للتحقق. النجاح يبقى تمثيل المورد. التشخيص الداخلي لا يدخل الجسم.

## ماذا بعد؟

هذا الشكل جزء من العقد الذي قد يتغير يومًا. الدرس التالي يضع الإصدار داخل التصميم لا داخل الديكور فقط.

## المصادر الرسمية

- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- NestJS — OpenAPI: https://docs.nestjs.com/openapi/introduction
