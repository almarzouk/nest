# catchError

## الهدف من هذا الدرس

التقاط فشل التيار داخل `pipe` وتحويله إلى خطأ HTTP مفهوم، أو تركه يكمل إلى المرشح عن قصد.

## المشكلة التي يحلها

طلب خارجي أو متحكم يرمي داخل التيار. من غير معامل، المرشح يرى الخطأ كما صدر. أحيانًا تريد ترجمة فشل الكتالوج الخارجي إلى `502` قبل أن يصل العميل رسالة مكتبة. `try/catch` حول `return next.handle().pipe(...)` لا يلتقط فشلًا يحدث لاحقًا عند الاشتراك.

## الفكرة ببساطة

`catchError` يستلم الخطأ ويعيد تيارًا بديلًا. إن أعدت `throwError(() => err)` فالخطأ يكمل. إن أعدت `of({ data: [] })` فقد ابتلعت الفشل وصار الرد نجاحًا. في API المتجر الفشل يبقى فشلًا، والترجمة تكون إلى `HttpException`.

## التعريف التقني

```ts
import { catchError, throwError } from 'rxjs';
import { BadGatewayException } from '@nestjs/common';

source.pipe(
  catchError((error: unknown) => {
    return throwError(() => new BadGatewayException('catalog upstream failed'));
  }),
);
```

`catchError` يجب أن يعيد `Observable`. `throwError` تيار يخطئ فور الاشتراك. رمي الاستثناء داخل الدالة يعمل أيضًا مع الإصدارات الحديثة لأنه يُحوَّل إلى خطأ تيار، لكن إرجاع `throwError` أوضح في القراءة.

لا تُرجع `of(null)` لخطأ مصادقة. ذلك `200` بجسم فارغ.

## Mental Model

```text
المصدر يخطئ
  catchError
    throwError(HttpException) -> المرشح -> رمز HTTP
    of(value)                 -> نجاح كاذب
```

## أبسط مثال

معترض كتالوج لا يبدّل النجاح، ويترجم أي فشل غير متوقع إلى بوابة سيئة عندما يكون المسار وكيلًا:

```ts
intercept(_context: ExecutionContext, next: CallHandler) {
  return next.handle().pipe(
    catchError((error: unknown) => {
      if (error instanceof HttpException) {
        return throwError(() => error);
      }
      return throwError(() => new BadGatewayException('catalog upstream failed'));
    }),
  );
}
```

`NotFoundException` التي رماها المتحكم تبقى كما هي. الخطأ الغريب يصبح `502`.

## شرح المثال

الفرع الأول يمرر أخطاء Nest التي تعرف رمزها. الفرع الثاني يمنع تسريب رسالة سائق أو مكدس إلى العميل. السجل التفصيلي مكانه `tap({ error })` قبل `catchError` أو داخل الفرع قبل `throwError`، لا داخل جسم `502`.

## ماذا يحدث داخليًا؟

الاشتراك يشغّل المصدر. عند `error` تتوقف `map` اللاحقة ولا يعمل `tap` على `next`. `catchError` يشترك في التيار البديل. `throwError` يصدر خطأ للمستمع الأعلى، وهو Nest، فيدخل `ExceptionFilter`. لذلك `try/catch` حول بناء `pipe` لا يرى `NotFoundException` التي تحدث داخل المتحكم عند التنفيذ. البناء نجح. الفشل جاء مع القيمة.

## مثال عملي

على مسار REST للكتب اترك مرشح Nest الافتراضي. ضع `catchError` حيث تترجم حدود النظام: فشل بحث ISBN خارجي ستستدعيه في الفصل التالي. داخل الخدمة، بعد `firstValueFrom`، يمكنك `try/catch` العادي لأن الوعد رفض. داخل المعترض الذي ما زال على التيار، الأداة `catchError`.

لا تلتقط الخطأ مرتين: مرة في الخدمة تحوله إلى `BadGatewayException` ومرة في المعترض تحوله إلى `500` عام فتفقد `502`.

## أخطاء شائعة

- `catchError` يعيد كائنًا عاديًا لا تيارًا، فينكسر الاشتراك.
- ابتلاع الخطأ بـ `of([])` وصفحة الرف تبدو فارغة والبحث كان معطلًا.
- `try/catch` حول `pipe` فقط.
- ترجمة `HttpException` المعروفة إلى `502`.

## متى نستخدمه؟

عند حدود تيار لا يملك المرشح سياقًا كافيًا: فشل منبع خارجي، أو إخفاء تفاصيل قبل الرمز.

## متى لا نستخدمه؟

لكل `NotFoundException` من خدمة الكتب. المرشح يحوّلها إلى `404` بلا معامل. ولا تخفِ أخطاء التحقق.

## مقارنة

| المعالجة | الرد الذي يراه العميل |
| --- | --- |
| بلا `catchError` | ما يصنعه المرشح من الخطأ الأصلي |
| `throwError(() => httpException)` | رمز الاستثناء |
| `of(fallback)` | نجاح بالجسم البديل |

## تمارين

### تمرين سهل

ماذا يجب أن يعيد `catchError`؟

<details>
<summary>الحل</summary>

`Observable`، مثل `throwError(...)` أو `of(...)`.

</details>

### تمرين متوسط

لماذا لا يلتقط `try/catch` حول `return next.handle().pipe(...)` خطأ المتحكم؟

<details>
<summary>الحل</summary>

لأن المتحكم يعمل عند الاشتراك لا عند بناء التيار. الخطأ يسافر داخل التيار.

</details>

### تمرين متقدم

متى يكون `of([])` خطأ منتج في رف الكتب؟

<details>
<summary>الحل</summary>

عندما يكون الفراغ ناتج عطل في المنبع لا ناتج بحث بلا كتب. العميل لا يفرّق بين الرف الفارغ والخدمة المعطلة.

</details>

## أسئلة مراجعة

- كيف تمرر `HttpException` بلا تغيير؟
- أي رمز يناسب منبع ISBN معطلًا؟
- أين `try/catch` كافٍ؟

## الخلاصة

`catchError` يبدّل تيار الخطأ. في المتجر ترجم الغريب وأعد رمي المعروف. لا تحوّل العطل إلى قائمة فارغة.

## ماذا بعد؟

`timeout`: إيقاف الانتظار عندما لا تصل القيمة أصلًا.

## المصادر الرسمية

- RxJS — `catchError`: https://rxjs.dev/api/index/function/catchError
- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
