# تحويلها إلى HTTP

## الهدف من هذا الدرس

ترجمة أصناف متجر رف إلى وضع HTTP وجسم مفهوم في فلتر واحد، من غير إعادة رمي `HttpException` كعطل 500.

## المشكلة التي يحلها

`BookNotFound` يصل إلى العميل كـ 500 لأن لا أحد يترجمه. محاولة الإصلاح تلتقط كل شيء بما فيه `NotFoundException` التي رماها حارس أو Pipe ثم تلفها في `InternalServerErrorException`. الغياب الحقيقي يصبح عطلًا، ويضيع جسم Nest الأصلي.

## الفكرة ببساطة

الفلتر على الحد يعرف الجدول: غياب 404، تعارض 409، قاعدة سعر 400، وغير ذلك يمر كما هو إن كان `HttpException`. الاستثناء المعروف لا يُعاد تغليفه.

## التعريف التقني

`@Catch(BookNotFound, IsbnTaken, InvalidShelfPrice)` يحدّد الأنواع. `ExceptionFilter` ينفّذ `catch(exception, host)`. `host.switchToHttp()` يعطي الاستجابة. `HttpException` التي يرميها الإطار (التحقق، `AuthGuard` من حزم Nest) تحمل `getStatus()` و `getResponse()`. الفلتر العالمي الذي يفعل `catch (e) { throw new InternalServerErrorException() }` أو يكتب 500 لكل ما يصل، يبتلع هذه الحالات. إن احتجت فلترًا عامًا، افحص `instanceof HttpException` وأعد `status` و `getResponse()` كما هما. لا تبنِ 500 جديدًا فوق 404.

## Mental Model

```text
BookNotFound        → 404
IsbnTaken           → 409
InvalidShelfPrice   → 400
HttpException       → status الأصلي
الباقي              → درس الأخطاء غير المتوقعة
```

## أبسط مثال

```ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpStatus } from '@nestjs/common';
import type { Response } from 'express';
import { BookNotFound } from '../books/books.errors.js';

@Catch(BookNotFound)
export class BookNotFoundFilter implements ExceptionFilter {
  catch(exception: BookNotFound, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse<Response>();
    res.status(HttpStatus.NOT_FOUND).json({
      statusCode: 404,
      message: 'Book not found',
      bookId: exception.bookId,
    });
  }
}
```

## شرح المثال

- الرسالة للعميل ثابتة ومفهومة. `bookId` من الصنف إن كان العقد يسمح بإظهاره لصاحب المستأجر.
- التسجيل يتم بـ `app.useGlobalFilters` في `main.ts` أو كمزوّد `APP_FILTER`.
- هذا الفلتر لا يرى `IsbnTaken`. إما توسّع `@Catch` أو تكتب فلتر مجال واحدًا بجدول، كما في المثال العملي.

## ماذا يحدث داخليًا؟

1. الاستثناء يخرج من الخدمة.
2. Nest يبحث عن فلتر يطابق النوع، من الأخص إلى الأعم.
3. الفلتر يكتب الوضع والجسم وينهي الاستجابة.
4. إن كان الاستثناء `HttpException` ولم يلتقطه فلترك، فلتر Nest المدمج يكتب وضعه. لا تضع فوقه فلترًا يعيد الكتابة إلى 500.
5. ترتيب الفلاتر العامة مهم: فلتر المجال قبل الفلتر الذي يتعامل مع المجهول.

## مثال عملي

```ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpException, HttpStatus } from '@nestjs/common';
import type { Response } from 'express';
import { BookNotFound, InvalidShelfPrice, IsbnTaken } from '../books/books.errors.js';

@Catch(BookNotFound, IsbnTaken, InvalidShelfPrice)
export class ShelfDomainFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse<Response>();
    if (exception instanceof HttpException) {
      res.status(exception.getStatus()).json(exception.getResponse());
      return;
    }
    const status = exception instanceof BookNotFound ? HttpStatus.NOT_FOUND
      : exception instanceof IsbnTaken ? HttpStatus.CONFLICT
      : HttpStatus.BAD_REQUEST;
    res.status(status).json({ statusCode: status, error: exception.constructor.name });
  }
}
```

الفرع الأول دفاع إن وُسّع `@Catch` لاحقًا. النقطة التي يطلبها هذا الدرس: لا يوجد سطر يبني `InternalServerErrorException` من `HttpException`. جسم الخطأ هنا مختصر؛ شكل العقد الكامل يُفصَّل في فصل تصميم الـ API حتى لا نخلط الترجمة بالشكل العام.

## أخطاء شائعة

- `try/catch` في المتحكم حول كل استدعاء، ثم `throw new InternalServerErrorException(error)`.
- فلتر `@Catch()` فارغ (يلتقط الكل) يكتب 500 دائمًا.
- إرجاع تفاصيل Prisma داخل JSON «للتشخيص».
- ترجمة `IsbnTaken` إلى 404 حتى «لا يعرف الناس أن الرقم مستخدم». هذا يكذب على العميل الذي يستطيع التصحيح.

## متى نستخدمه؟

عندما تخرج أصناف المجال من الخدمة وتريد رمزًا واحدًا لكل صنف، مثبتًا باختبار e2e على المسار.

## متى لا نستخدمه؟

داخل الـ Service. ولا لتغيير رمز `ValidationPipe`. الأنبوب يكتب 400 قبل الخدمة.

## مقارنة

| القادم | الوضع |
| --- | --- |
| `BookNotFound` | 404 |
| `IsbnTaken` وتعارض الحالة | 409 |
| `InvalidShelfPrice` | 400 |
| `HttpException` قائمة | `getStatus()` بلا تغليف |

## تمارين

### تمرين سهل

التحقق فشل لأن `title` غائب. هل يمر على `ShelfDomainFilter`؟

<details>
<summary>الحل</summary>

لا، إن كان الفلتر لا يلتقط `BadRequestException`. `ValidationPipe` يرمي `HttpException` ويكتبها فلتر Nest. تدخلك لتغليفها هو الذي يحوّلها إلى 500 بالخطأ.

</details>

### تمرين متوسط

الكود التالي في فلتر عام: `catch (e) { if (e instanceof HttpException) throw new InternalServerErrorException(e.message) }`. ما الرمز النهائي لكتاب غائب رماه `NotFoundException` من مكان آخر؟

<details>
<summary>الحل</summary>

500. هذا هو الخطأ الممنوع. المطلوب إعادة `e.getStatus()` و `e.getResponse()`.

</details>

### تمرين متقدم

اختبار يطلب `POST /books/:id/isbn` برقم مأخوذ ويتوقع 409 وجسمًا بلا SQL. أين تضع التوقع: وحدة الخدمة أم e2e؟

<details>
<summary>الحل</summary>

وحدة الخدمة تتوقع `IsbnTaken`. e2e على التطبيق مع الفلتر المسجّل يتوقع 409 والجسم. تقسيم الاختبار يمنع ترجمة «ناجحة» في الوحدة بينما المسار ما زال 500.

</details>

## أسئلة مراجعة

- أين تُكتب الترجمة: خدمة أم فلتر؟
- ماذا تفعل حين يكون الاستثناء `HttpException` أصلًا؟
- أي رمز لـ `IsbnTaken`، وأي رمز لسعر سالب في المجال؟
- لماذا `@Catch()` الواسع أخطر من `@Catch(BookNotFound)`؟

## الخلاصة

جدول صغير في فلتر الحد يحوّل أصناف رف إلى 404 و 409 و 400. `HttpException` تُعاد كتابتها بوضعها الأصلي. تغليفها في 500 يمحو معنى الغياب والتحقق.

## ماذا بعد؟

تبقى الأخطاء التي لا تعرف اسمها: سقوط القاعدة والأخطاء البرمجية. الدرس التالي يقرر ماذا يرى العميل وماذا يُكتب في السجل.

## المصادر الرسمية

- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- MDN — 404: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/404
- MDN — 409: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/409
