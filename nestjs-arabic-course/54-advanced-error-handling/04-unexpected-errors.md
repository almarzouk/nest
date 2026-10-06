# الأخطاء غير المتوقعة

## الهدف من هذا الدرس

عزل العطل الذي لا يملك صنف مجال في متجر رف: العميل يرى 500 بلا أحشاء القاعدة، والسجل يرى السبب وسلسلة الاستدعاء.

## المشكلة التي يحلها

انقطع Postgres أثناء `publish`. الفلتر كتب رسالة `ECONNREFUSED` وكود Prisma في JSON. أو العكس: التقط المتحكم الاستثناء وابتلعَه وأعاد `{ ok: true }`. في الحالتين لا تعرف ماذا تصلح، والعميل إما رأى سرًا أو ظن أن الكتاب نُشر.

## الفكرة ببساطة

غير المتوقع يعني: العملية لم تكتمل، والعميل لا يستطيع إصلاح الجسم. الرد 500 عام. التفاصيل تذهب إلى السجل مع معرّف طلب. لا تُعد المحاولة من داخل `catch` يصمت.

## التعريف التقني

فلتر `@Catch()` بلا وسائط يلتقط ما لم يلتقطه فلتر أخص. أول سطر فيه: إن كان `exception instanceof HttpException` فاكتب `getStatus()` و `getResponse()` وتوقف. ما تبقى يُسجَّل بمستوى error عبر `Logger` من `@nestjs/common`، ثم تُكتب استجابة 500 بجسم ثابت مثل `{ statusCode: 500, message: 'Internal server error' }`. لا تمرر `exception` إلى `json`. صفحة Exception filters هي مكان `BaseExceptionFilter` إن أردت السلوك المدمج ثم تضيف السجل. معرّف الارتباط، إن وُجد لاحقًا في فصل المراقبة، يُضاف إلى سطر السجل لا إلى سر الاتصال.

## Mental Model

```text
HttpException      → اترك رمزه
صنف مجال مُلتقط    → فلتر المجال سبقك
أي شيء آخر         → log + 500 ثابت
```

## أبسط مثال

```ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpException, HttpStatus, Logger } from '@nestjs/common';
import type { Response } from 'express';

@Catch()
export class UnknownErrorFilter implements ExceptionFilter {
  private readonly logger = new Logger(UnknownErrorFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    if (exception instanceof HttpException) {
      const res = host.switchToHttp().getResponse<Response>();
      res.status(exception.getStatus()).json(exception.getResponse());
      return;
    }
    this.logger.error(exception instanceof Error ? exception.stack : exception);
    host.switchToHttp().getResponse<Response>().status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      statusCode: 500,
      message: 'Internal server error',
    });
  }
}
```

## شرح المثال

- الفرع الأول هو الحارس ضد تحويل 404 إلى 500.
- `stack` في السجل لا في الجسم.
- الرسالة الثابتة لا تتضمن اسم الجدول ولا سلسلة الاتصال التي قد تظهر في `Error.message` لمشغلات بعض القواعد.

## ماذا يحدث داخليًا؟

1. Prisma يرفض الوعد بخطأ تشغيل.
2. لا فلتر مجال يطابق النوع، فيصل إلى `@Catch()`.
3. `logger.error` يكتب أثناء معالجة الاستثناء. لا تنتظر `await` شبكة تسجيل إن كانت قد تسقط هي أيضًا؛ السجل المحلي يكفي هنا.
4. الاستجابة تُغلق بـ 500. الأثر الذي لم يُلتزم في المعاملة يجب أن يتراجع داخل الخدمة (المعاملة) لا في الفلتر.
5. العميل يعيد المحاولة فقط إن كانت العملية آمنة للإعادة. 500 ليس إذنًا عامًا بإعادة الدفع.

## مثال عملي

اختبار الفلتر يمرر خطأً عاديًا ويتأكد أن الجسم لا يحتوي كلمة `prisma`:

```ts
import { HttpException } from '@nestjs/common';

export function publicBody(exception: unknown) {
  if (exception instanceof HttpException) {
    return { status: exception.getStatus(), body: exception.getResponse() };
  }
  return { status: 500, body: { statusCode: 500, message: 'Internal server error' } };
}
```

استدعِ `publicBody(new HttpException('Book not found', 404))` وتوقع 404. استدعِ `publicBody(new Error('prisma://user:pass@host'))` وتوقع الجسم الثابت. هذا يثبت فرع الحماية وفرع العتمة معًا.

## أخطاء شائعة

- `res.status(500).json({ message: exception.message })` فيفضح SQL.
- `catch` فارغ في الخدمة يعيد الكتاب القديم كأنه التحديث نجح.
- تسجيل `error` لكل 404 بعد أن عطلت فحص `HttpException`.
- إرجاع 200 مع `{ error: 'failed' }` لأن الواجهة «لا تحب» 500.

## متى نستخدمه؟

كشبكة أخيرة بعد فلاتر الأنواع المعروفة. كل مسار يمر منها إن سقط شيء لم تسمِّه.

## متى لا نستخدمه؟

بدل أنواع المجال. إن صار كل فشل 500، فالجدول في الدرس السابق غير موصول. ولا كمكان لإعادة محاولة الدفع تلقائيًا.

## مقارنة

| | العميل | السجل |
| --- | --- | --- |
| `HttpException` | الوضع الأصلي | بلا ضجيج error إن كان 4xx متوقعًا |
| صنف مجال | حسب جدول الترجمة | info أو warn حسب المنتج |
| غير متوقع | 500 ثابت | error مع stack |

## تمارين

### تمرين سهل

رسالة الخطأ `password authentication failed`. هل تظهر في JSON؟

<details>
<summary>الحل</summary>

لا. تظهر في السجل فقط. الجسم `{ statusCode: 500, message: 'Internal server error' }`.

</details>

### تمرين متوسط

الفلتر العام سجّل كل `NotFoundException` كـ error وحوّل بعضها إلى 500. أي شرط ناقص؟

<details>
<summary>الحل</summary>

`instanceof HttpException` في أول `catch`، ثم إرجاع `getStatus()` و `getResponse()` من غير `Logger.error` لهذه الحالات.

</details>

### تمرين متقدم

النشر يكتب صف الكتاب ثم يرسل بريدًا، والبريد يرمي. ماذا يثبت في القاعدة، وماذا يرى العميل؟

<details>
<summary>الحل</summary>

إن كان البريد بعد الالتزام، فالكتاب منشور والعميل قد يرى 500 رغم النجاح. إما تُدخل الأثر الجانبي بعد الرد عبر مهمة، أو تكون المعاملة لم تُلتزم بعد والفشل يتراجع. الفلتر لا يلغي `UPDATE` نُفّذ. هذا حد الفلتر: هو شكل الرد والسجل، لا تعويض العمل.

</details>

## أسئلة مراجعة

- ماذا يرى العميل عند انقطاع القاعدة؟
- أين يذهب `stack`؟
- كيف تحمي `HttpException` داخل `@Catch()` العام؟
- لماذا ابتلاع الخطأ وإرجاع نجاح أسوأ من 500؟

## الخلاصة

غير المتوقع 500 بجسم ثابت وسجل كامل. `HttpException` لا تدخل هذا المسار. الفلتر لا يصلح الأثر الذي التزم به في القاعدة.

## ماذا بعد؟

القطع الثلاث — تصنيف، أصناف، ترجمة، وعطل — تحتاج ترتيبًا واحدًا يلتزمه كل مسار. الدرس التالي يجمعها في استراتيجية لا في فلاتر متنافسة.

## المصادر الرسمية

- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- NestJS — Logger: https://docs.nestjs.com/application/logger
