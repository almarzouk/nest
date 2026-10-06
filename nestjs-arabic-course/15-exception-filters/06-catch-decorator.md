# @Catch

## الهدف من هذا الدرس

تحديد أي استثناءات يدخل الفلتر عبر `@Catch`، وتجنّب تحويل `400` إلى `500`.

## المشكلة التي يحلها

فلتر بلا أنواع يمسك كل شيء، بما فيه `BadRequestException` و `SeatTakenException`، ثم يكتب `500`. العميل يفقد الفرق بين مقعد تالف ومقعد مبيع وعطل خادم.

## الفكرة ببساطة

`@Catch(SeatTakenException)` يمسك ذلك الصنف فقط. `@Catch(HttpException)` يمسك الأب والفئات. `@Catch()` بلا وسيط يمسك الكل. في الفراغ افحص النوع: إن كان `HttpException` فانسخ `getStatus()` و `getResponse()`، وإن كان غيره فاكتب `500` من غير جسم داخلي.

## التعريف التقني

الزينة من `@nestjs/common` على صنف الفلتر. يمكن تمرير أكثر من نوع: `@Catch(SeatTakenException, NotFoundException)`.

القوس الفارغ يعني أي قيمة رُميت، ومنها `TypeError`. هذا مفيد لسجل أخير عام، وخطير إن كان الجسم واحدًا لكل الأنواع.

نمط آمن للفراغ:

```ts
const status = exception instanceof HttpException ? exception.getStatus() : HttpStatus.INTERNAL_SERVER_ERROR;
const body = exception instanceof HttpException ? exception.getResponse() : { statusCode: status, message: 'Internal server error' };
```

`getResponse()` قد يكون نصًا أو كائنًا. إن كان كائن `SeatTakenException` ففيه `code`. لا تستبدله بـ `{ message: 'Internal server error' }`.

لا ترمِ `new InternalServerErrorException()` من فرع `HttpException`. ذلك تحويل `400` و `409` إلى `500`.

الفلتر المربوط بهذا `@Catch` لا يرى نوعًا خارج القائمة. ذلك النوع يكمل إلى فلتر أوسع إن وُجد في مستوى لاحق حسب ترتيب الحل. داخل المستوى نفسه أول تطابق يمسك.

## Mental Model

```text
@Catch(SeatTakenException)   409 ذو code فقط
@Catch(HttpException)        كل HTTP بما فيه 400
@Catch()                     الكل، مع فرع يحفظ رمز HTTP
```

## أبسط مثال

```ts
import { Catch, type ExceptionFilter, HttpException, HttpStatus, type ArgumentsHost } from '@nestjs/common';
import type { Response } from 'express';

@Catch()
export class DeskExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();
    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      const payload = exception.getResponse();
      response.status(status).json(
        typeof payload === 'string' ? { statusCode: status, message: payload } : payload,
      );
      return;
    }
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
      message: 'Internal server error',
    });
  }
}
```

`SeatTakenException` يخرج بجسمه و `409`. `TypeError` يخرج `500` عامًا بلا مكدس.

## شرح المثال

- `instanceof` قبل أي كتابة ثابتة.
- النص يُلف ليبقى `statusCode`. الكائن يُمرَّر كما بُني، وفيه `code` إن وُجد.
- فرع المجهول لا ينسخ `exception` إلى JSON.
- `@Catch()` هنا عام الخطر إن حذفت الفرع. الفرع هو الدرس.

## ماذا يحدث داخليًا؟

Nest تقارن الاستثناء بقائمة `@Catch`. التطابق يستدعي `catch` ويتوقف. `@Catch(HttpException)` يطابق الفئات لأن `instanceof HttpException` ينجح لها. `@Catch(SeatTakenException)` لا يطابق `NotFoundException`. ذلك الاستثناء يبحث عن فلتر آخر في ترتيب المسار ثم المتحكم ثم العام.

فلتر `@Catch()` على المسار يبتلع الكل قبل أن يصل العام، حتى أخطاء البرمجة. إن كان يسجل ثم يعيد الرمز الصحيح فذلك مفيد. إن كان يكتب `500` دائمًا فأنبوب التحقق «ينكسر» من جهة العميل.

لا تمرر الاستثناء إلى فلتر ثانٍ بنداء يدوي. الوراثة من فلتر أساس إن أردت مشاركة كتابة الرأس.

## مثال عملي

الشباك يضع `DeskExceptionFilter` عامًا بهذا الفحص. اختبار `seat: 0` يتوقع `400` لا `500`. اختبار المقعد المبيع يتوقع `code: SEAT_TAKEN` و `409`. اختبار رمي `TypeError` من مسار تجريبي يتوقع `500` وغياب `stack`. حذف فرع `instanceof` يكسر الاختبار الأول، وهو دليل المراجعة.

## أخطاء شائعة

- `@Catch()` ثم `status(500)` دائمًا.
- `@Catch(Error)` لالتقاط «المجهول فقط». سلسلة `HttpException` تصل إلى `Error` عبر `IntrinsicException`، فيمسك القوس أخطاء HTTP أيضًا. المجهول يُفرز داخل `@Catch()` بفحص `instanceof HttpException`.
- ابتلاع `HttpException` داخل الفلتر من غير كتابة رد، فيعلق العميل.
- تسجيل `getResponse()` كاملًا إن كان يومًا فيه بيانات حساسة. عقد `SeatTakenException` لا يضع أسرارًا.

## متى نستخدمه؟

`@Catch(نوع)` لسجل أو شكل يخص حالة واحدة. `@Catch(HttpException)` لتوحيد غلاف مع الإبقاء على الرمز. `@Catch()` لشبكة أمان أخيرة بشرط فرع HTTP.

## متى لا نستخدمه؟

لا تفرغ القوس ثم توحّد الرمز على `500`. لا تلتقط نوعًا حتى تخفي `409` عن الواجهة التي تتفرع على `code`.

## مقارنة

| الزينة | تمسك | خطر |
| --- | --- | --- |
| `@Catch(SeatTakenException)` | تلك الحالة | ضيق مقصود |
| `@Catch(HttpException)` | `400` و `401` و `403` و `404` و `409` وغيرها | كتابة `500` ثابت |
| `@Catch()` | المجهول أيضًا | طمس الرمز إن لم تفحص |

## تمارين

### تمرين سهل

كيف تجعل الفلتر يرى `SeatTakenException` فقط؟

<details>
<summary>الحل</summary>

`@Catch(SeatTakenException)` على الصنف.

</details>

### تمرين متوسط

`@Catch()` استقبل `BadRequestException`. أي رمز تكتب في `response.status`؟

<details>
<summary>الحل</summary>

`400` عبر `exception.getStatus()`، لا `500`.

</details>

### تمرين متقدم

فلتر المسار `@Catch()` يكتب دائمًا `Internal server error` برمز `500`. لماذا فشل اختبار الزينة، وأين الإصلاح؟

<details>
<summary>الحل</summary>

فلتر المسار أولى من العام ويمسك `BadRequestException` قبل أن يخرج جسم الأنبوب. الإصلاح: فرع `instanceof HttpException` ينسخ الحالة والجسم، ويبقي `500` للمجهول فقط.

</details>

## أسئلة مراجعة

- ماذا يمسك `@Catch()` الفارغ؟
- كيف تحافظ على `409` داخل فلتر واسع؟
- لماذا لا يُمرَّر الاستثناء إلى الفلتر التالي؟
- لماذا لا تعتمد على `@Catch(Error)` لفئات HTTP؟

## الخلاصة

`@Catch` يضيّق الأنواع. القوس الفارغ مسموح لشبكة أخيرة إن نسخت رمز `HttpException` وجسمه. تثبيت `500` يحوّل رفض العميل إلى عطل خادم.

## ماذا بعد؟

الدرس التالي تسجيل الفلتر على التطبيق كله، ومن يرى استثناء Middleware.

## المصادر الرسمية

- [Exception filters](https://docs.nestjs.com/exception-filters)
- [Request lifecycle](https://docs.nestjs.com/faq/request-lifecycle)
