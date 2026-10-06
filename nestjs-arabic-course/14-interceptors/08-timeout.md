# Timeout

## الهدف من هذا الدرس

إيقاف انتظار الرد حين لا يصدر المعالج قيمة خلال مدة، وإرجاع `RequestTimeoutException` بدل ترك العميل معلقًا.

## المشكلة التي يحلها

تقرير مبيعات الشباك ينتظر استعلامًا ثقيلًا. العميل يعلّق حتى مهلة المتصفح. السجل لا يميّز التعليق عن النجاح البطيء. عامل `timeout` على التيار يفشل الوصف بعد المدة حتى يرد الإطار.

## الفكرة ببساطة

`next.handle().pipe(timeout(5000), catchError(...))`. إن انتهت المدة قبل الإصدار، RxJS يخطئ بـ `TimeoutError`. المعترض يحوّله إلى `RequestTimeoutException` برمز `408`. أي خطأ آخر يُعاد رميه كما هو حتى لا يصبح فشل المقعد `408`.

## التعريف التقني

من صفحة المعترضات، العاملان `timeout` و `catchError`، والاستثناء `RequestTimeoutException`، والنوع `TimeoutError` من `rxjs`.

```ts
return next.handle().pipe(
  timeout(5000),
  catchError((err: unknown) => {
    if (err instanceof TimeoutError) {
      return throwError(() => new RequestTimeoutException());
    }
    return throwError(() => err);
  }),
);
```

الصفحة تحذّر: إلغاء الاشتراك في التيار لا يوقف عملًا بدأ داخل الدالة، مثل استعلام قاعدة ما زال واعدًا. `408` للعميل لا يلغي الأثر الجانبي وحده. إن احتجت إلغاء الاستعلام، مرّر إشارة إلغاء داخل الخدمة. ذلك خارج هذا المعترض.

المدة في المثال الرسمي 5000 ملي ثانية. للتذاكر الحية قد تكون أقصر، ولتقرير ليلي أطول. المدة ثابت المعترض أو قيمة تُحقن، لا رقم سحري داخل الخدمة.

`timeout` عامل خروج على التيار: الساعة تبدأ حين يُشترَك، أي حين تبدأ البقية بالعمل.

## Mental Model

```text
handle يبدأ الاستعلام
  إصدار قبل 5ث → القيمة تمر ولا تدخل فرع TimeoutError
  لا إصدار → TimeoutError → 408
  BadRequest → يُعاد رميه 400
```

## أبسط مثال

```ts
import {
  type CallHandler,
  type ExecutionContext,
  Injectable,
  type NestInterceptor,
  RequestTimeoutException,
} from '@nestjs/common';
import { type Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class ReportTimeoutInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout(5000),
      catchError((err: unknown) => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}
```

اربطه على `GET /reports/sales` لا على كل `POST /tickets`. الحجز السريع لا يحتاج مهلة التقرير.

## شرح المثال

- `timeout(5000)` من `rxjs/operators`، و `TimeoutError` من `rxjs`.
- `throwError(() => err)` يحافظ على `BadRequestException` و `ForbiddenException`.
- تحويل كل خطأ إلى `RequestTimeoutException` يخفي `400`. الشرط `instanceof` هو الدرس.
- لا `map`. الجسم الناجح يبقى كما رجع التقرير.

## ماذا يحدث داخليًا؟

الاشتراك يبدأ الساعة. إن صدرت القيمة، `timeout` يمررها ويلغي الساعة. إن سبق الخطأ المدة، `catchError` يرى ذلك الخطأ لا `TimeoutError`. إن سكت التيار حتى المدة، يُنشأ `TimeoutError` ثم يبدله المعترض بـ `RequestTimeoutException`. الفلتر الافتراضي يكتب `408`.

الدالة قد تكون ما زالت عند `await` الاستعلام. الرد للعميل انتهى من جهة هذا الطلب، والاستعلام قد يكمل ويكتب صفًا. لذلك المهلة على معترض تقرير للقراءة أسلم من مهلة على `POST` الذي يحفظ، إلا إذا ألغت الخدمة العمل.

حارس فشل قبل المعترض لا يبدأ الساعة. أنبوب فشل بسرعة يُعاد رميه ولا ينتظر خمس ثوانٍ.

## مثال عملي

`SalesReportController` عليه المعترض. تقرير يرجع خلال ثانية يصل كما هو. تقرير ينام ست ثوانٍ في الاختبار المزيف يرجع `408`. طلب التقرير بمعامل صفحة تالف يرجع `400` فورًا لأن الخطأ ليس `TimeoutError`. اختبار الوحدة يمرر `CallHandler` لا يصدر أبدًا ويتقدم الساعة الوهمية، أو ينتظر مدة قصيرة في اختبار أبطأ على مهلة 20 ملي ثانية في نسخة اختبار من المعترض.

## أخطاء شائعة

- `catchError(() => throwError(() => new RequestTimeoutException()))` بلا تمييز، فيصبح كل `400` مهلة.
- الاعتقاد أن `408` يرجّع معاملة قاعدة أُرسلت.
- مهلة خمس ثوانٍ على الصحة فتفشل الفحوص البطيئة أصلًا بلا داعٍ.
- `timeout` قبل أن تفهم أن الساعة على التيار لا على Middleware.

## متى نستخدمه؟

على عمليات قراءة قد تطول، بمدة معروفة، مع إعادة رمي أي خطأ ليس `TimeoutError`.

## متى لا نستخدمه؟

على كتابة مالية من غير إلغاء في الخدمة. وعلى أخطاء الشكل. ولا كبديل عن فهرس الاستعلام إن كان التقرير بطيئًا دائمًا.

## مقارنة

| الخطأ الداخل | ماذا يخرج من المعترض |
| --- | --- |
| قيمة في الوقت | القيمة نفسها |
| `TimeoutError` | `RequestTimeoutException` أي `408` |
| `BadRequestException` | نفسه |
| رفض حارس | المعترض لم يبدأ |

## تمارين

### تمرين سهل

أي استثناء HTTP يقابل انتهاء المهلة في هذا الدرس؟

<details>
<summary>الحل</summary>

`RequestTimeoutException` ورمزه `408`.

</details>

### تمرين متوسط

لماذا يُعاد `throwError(() => err)` في الفرع الآخر؟

<details>
<summary>الحل</summary>

حتى لا تتحول أخطاء الشكل والإذن والأعمال إلى `408`. المهلة حالة واحدة فقط.

</details>

### تمرين متقدم

العميل رأى `408` وظهر صف تقرير جديد في الجدول. ما الذي لم يفعله المعترض؟

<details>
<summary>الحل</summary>

لم يوقف العمل داخل الدالة. إلغاء الاشتراك لا يلغي `await` على قاعدة البيانات. الإلغاء يُمرَّر إلى الخدمة إن كان الأثر يجب أن يتوقف.

</details>

## أسئلة مراجعة

- أي عامل يحد المدة؟
- كيف تميّز المهلة عن `400`؟
- هل يُلغى استعلام القاعدة تلقائيًا؟
- لماذا لا يُربَط المعترض على كل المسارات في المثال؟

## الخلاصة

`timeout` يفشل التيار الصامت، و `catchError` يحوّل `TimeoutError` فقط إلى `408` ويعيد بقية الأخطاء. الرد للعميل لا يعني أن العمل داخل الدالة توقف.

## ماذا بعد؟

الدرس التالي معاينة تخزين: إرجاع تيار جاهز بـ `of` من غير استدعاء `handle` حين تكون القيمة محفوظة.

## المصادر الرسمية

- [Interceptors](https://docs.nestjs.com/interceptors)
