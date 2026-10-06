# استراتيجية واحدة

## الهدف من هذا الدرس

جمع التصنيف وأصناف المجال والترجمة والعطل في مسار واحد لمتجر رف، بحيث يضيف مبرمج ميزة جديدة صفًا في الجدول لا فلترًا خاصًا به.

## المشكلة التي يحلها

مسار ISBN يستخدم فلترًا، ومسار السعر يعيد `BadRequestException` من الخدمة، ومسار الحذف يلتقط الكل في المتحكم. المراجعة لا تستطيع الإجابة: «ماذا يرى العميل إن غاب الكتاب؟» من غير فتح كل ملف.

## الفكرة ببساطة

وثيقة قصيرة في الكود: هذه الأصناف، هذا الرمز، هذا الجسم، وهذا ما يُسجَّل. الخدمة ترمي صنفًا. فلتر مجال وفلتر ختامي يطبّقان الوثيقة. المتحكم لا يملك `try/catch`.

## التعريف التقني

سجّل الفلاتر بحيث يُسأل الأخص أولًا. في Nest، الفلتر العام الأخير هو شبكة المجهول، والفلتر المسجّل لنوع محدد يعالج أصنافه. `APP_FILTER` مع `useClass` يربط الفلتر بالحقن إن احتاج `Logger` أو معرّف طلب. لا تكرر الجدول داخل `BooksController`. اختبار e2e واحد لكل صف في الجدول يثبت الاستراتيجية. الاستراتيجية لا تغيّر نجاح القائمة الفارغة: ذلك 200، ويُذكر هنا حتى لا «يوحّد» أحدهم الغياب بجعل كل فراغ 404.

## Mental Model

```text
Pipe → 400 جاهز
Guard → 401/403 جاهز
Service → صنف مجال
ShelfFilter → 400/404/409
UnknownFilter → HttpException كما هي، والباقي 500 + log
```

## أبسط مثال

```text
BookNotFound         404   لا SQL في الجسم
IsbnTaken            409
InvalidShelfPrice    400
غير ذلك              500   stack في السجل فقط
HttpException        لا تُغلَّف
```

## شرح المثال

- أربعة صفوف تكفي بداية رف. صف جديد يعني صنفًا جديدًا وسطرًا في الفلتر واختبارًا.
- «لا تُغلَّف» سطر في الاستراتيجية لا تعليق في مراجعة واحدة.
- القائمة `GET /books` التي تعيد `[]` ليست في الجدول لأنها ليست فشلًا.

## ماذا يحدث داخليًا؟

1. الطلب يدخل الأنابيب والحراس. فشلهم `HttpException` يكتبها الإطار إن لم تلتقطها شبكتك خطأً.
2. الخدمة إما تعيد كتابًا أو ترمي صنفًا من `books.errors.ts`.
3. `ShelfDomainFilter` يطابق `@Catch` ويكتب الرمز.
4. ما لم يُطابق يصل إلى `UnknownErrorFilter`.
5. إن أضاف أحد `try/catch` في المتحكم ورمى `InternalServerErrorException`، فقد كسر الخطوة 3 قبل أن ترى الفلتر الصنف. راجع المراجعات على هذا النمط.

## مثال عملي

```ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpStatus } from '@nestjs/common';
import type { Response } from 'express';
import { BookNotFound, InvalidShelfPrice, IsbnTaken } from '../books/books.errors.js';

const table = new Map<Function, number>([
  [BookNotFound, HttpStatus.NOT_FOUND],
  [IsbnTaken, HttpStatus.CONFLICT],
  [InvalidShelfPrice, HttpStatus.BAD_REQUEST],
]);

@Catch(BookNotFound, IsbnTaken, InvalidShelfPrice)
export class ShelfStrategyFilter implements ExceptionFilter {
  catch(exception: object, host: ArgumentsHost) {
    const status = table.get(exception.constructor) ?? HttpStatus.INTERNAL_SERVER_ERROR;
    host.switchToHttp().getResponse<Response>().status(status).json({
      statusCode: status,
      message: status === 404 ? 'Book not found' : status === 409 ? 'Conflict' : 'Invalid shelf rule',
    });
  }
}
```

إضافة `BookWithdrawn` تعني سطرًا في `table` وإضافة النوع إلى `@Catch`. الفلتر الختامي من الدرس السابق يبقى مسجّلًا بعده ويحمي `HttpException`. جسم الرسائل هنا ثابت عن قصد؛ توحيد شكل الحقول (`code`, `message`) يُستكمل في درس شكل الخطأ دون أن يتعدد أسلوب الترجمة.

## أخطاء شائعة

- استراتيجيتان: خريطة في الفلتر و `if` في كل متحكم.
- اعتبار 500 مسارًا مقبولًا لخطأ مجال نسيت إضافته إلى `@Catch`. راقب السجل: ظهور اسم الصنف داخل 500 يعني صفًا ناقصًا.
- توحيد قاسٍ يجعل `GET /books` الفارغ 404 لأنه «لا بيانات».
- نسخ الفلتر لكل وحدة كتب ومؤلفين حتى يتباعد الجدولان.

## متى نستخدمه؟

من أول مسار يغيّر حالة في رف، قبل أن تتعدد الميزات. كل ميزة تلحق بالجدول.

## متى لا نستخدمه؟

لا تبنِ لغة أخطاء عامة للمنتج كله في أسبوع فارغ. ابدأ بأصناف الكتب الحقيقية ووسّع الجدول حين يظهر نوع جديد.

## مقارنة

| الأسلوب | إضافة قاعدة نشر جديدة |
| --- | --- |
| `try/catch` في المتحكم | تعديل المتحكم واختراع رمز |
| `HttpException` من الخدمة | الخدمة تعرف HTTP والاختبار أيضًا |
| جدول في فلتر واحد | صنف + سطر + اختبار e2e |

## تمارين

### تمرين سهل

أين يُلتقط فشل `ValidationPipe` ضمن الاستراتيجية؟

<details>
<summary>الحل</summary>

في سلوك `HttpException` المدمج، ويمر من الفلتر الختامي إن كان عامًا بشرط ألا يغيّر الوضع. ليس صفًا في `table` إلا إذا قررت شكلًا خاصًا للتحقق، وهذا استثناء يُكتب في الوثيقة.

</details>

### تمرين متوسط

مطور أضاف `BookArchived extends Error` ونسي `@Catch`. ماذا يرى العميل، وماذا تفعل المراجعة؟

<details>
<summary>الحل</summary>

500 من الفلتر الختامي، واسم الصنف في السجل. المراجعة ترفض الدمج حتى يدخل الصنف الجدول والاختبار الذي يتوقع 409 أو الرمز المتفق عليه.

</details>

### تمرين متقدم

اكتب ثلاثة اختبارات e2e تثبت الاستراتيجية دفعة واحدة.

<details>
<summary>الحل</summary>

1. معرّف غائب → 404 والجسم بلا `stack`.
2. ISBN مكرر → 409.
3. مسار يرمي `NotFoundException` من حارس موارد → 404 لا 500، ليثبت أن الفلتر العام لا يغلّف `HttpException`.

</details>

## أسئلة مراجعة

- ما مدخل الاستراتيجية وما مخرجها؟
- لماذا المتحكم بلا `try/catch`؟
- كيف تكشف صنف مجال نسيه الجدول؟
- أي نجاح يُستثنى من جدول الفشل عمدًا؟

## الخلاصة

استراتيجية رف جدول واحد: الخدمة ترمي صنفًا، الفلتر يكتبه، المجهول يصبح 500 مع سجل، و `HttpException` تبقى على رمزها. الميزة الجديدة سطر في الجدول لا أسلوب جديد.

## ماذا بعد؟

الرمز الذي تكتبه الاستراتيجية سيعيش مع العملاء طويلًا. الفصل التالي يشغّل أكثر من نسخة للمسارات داخل التطبيق نفسه عندما يتغير العقد.

## المصادر الرسمية

- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- NestJS — Validation: https://docs.nestjs.com/application/validation
