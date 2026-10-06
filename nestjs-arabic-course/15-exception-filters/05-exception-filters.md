# Exception Filters

## الهدف من هذا الدرس

كتابة صنف يطبّق `ExceptionFilter` ويشكّل الرد من الاستثناء والمضيف.

## المشكلة التي يحلها

الجسم الافتراضي لا يطابق عقد الشباك: تريد `path` و `code` معًا، أو تريد تسجيل `cause` ثم كتابة JSON بأنفسكم. الفلتر المدمج لا يفتح لك هذه الخطوة إلا إذا استبدلته أو أحطت به لأنواع معينة.

## الفكرة ببساطة

`catch(exception, host)` تقرأ الطلب والرد من `host.switchToHttp()` وتكتب الحالة والجسم. الصنف لا يُستدعى إلا لاستثناء غير ممسوك من النوع الذي ستربطه بـ `@Catch` في الدرس التالي. هنا شكل الدالة.

## التعريف التقني

```ts
interface ExceptionFilter<T = any> {
  catch(exception: T, host: ArgumentsHost): void;
}
```

`ArgumentsHost` هو أب `ExecutionContext`. في HTTP:

```ts
const ctx = host.switchToHttp();
const response = ctx.getResponse<Response>();
const request = ctx.getRequest<Request>();
```

أنواع `Response` و `Request` من Express حين يكون المحوّل Express. على Fastify الواجهة تختلف. اكتب ما يطابق المحوّل أو استخدم أساليب الرد التي يوثّقها Nest لمحوّلك.

`host.getType()` يميّز HTTP عن غيره قبل `switchToHttp()` إن كان الفلتر عامًا على تطبيق هجين.

الفلتر يكتب الرد. لا يرجع التيار كما المعترض. بعد `catch` يُعتبر الاستثناء ممسوكًا. فلتر آخر لا يراه. الصفحة تقول إن الاستثناء لا يُمرَّر من فلتر إلى فلتر. التشابه بين الفلاتر يتم بالوراثة لا بسلسلة `next`.

لا تستدعِ `response.json` ثم ترمي من جديد إلا وأنت تقصد رميًا بعد ما قد كُتب جزء من الرد. اكتب مرة واحدة.

`BaseExceptionFilter` من `@nestjs/core` يغلّف سلوك المدمج إن أردت السجل ثم `super.catch`. هذا الدرس يكتب الرد بنفسه حتى ترى الحقول.

## Mental Model

```text
استثناء غير ممسوك
  catch
    اقرأ request.url
    اكتب status + JSON
  لا فلتر تالٍ
```

## أبسط مثال

```ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpException } from '@nestjs/common';
import type { Request, Response } from 'express';

@Catch(HttpException)
export class TicketHttpFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    response.status(status).json({
      statusCode: status,
      path: request.url,
      error: exception.message,
    });
  }
}
```

`@Catch` ظاهر لأن الربط من غيره لا يوصل الاستثناء إلى الصنف. معنى القوسين درس تالٍ. هنا انظر إلى `catch`.

## شرح المثال

- الحالة تُنسخ من الاستثناء لا تُثبَّت على `500`.
- `path` يساعد الواجهة في السجل من غير تسريب مكدس.
- `exception.message` قد يكون عامًا. `getResponse()` يعيد النص أو الكائن الذي بُني به الاستثناء إن احتجت `code` من `SeatTakenException`.
- الصنف لا يفحص المقعد. هو آخر خطوة على السلك.

## ماذا يحدث داخليًا؟

عند الرمي غير الممسوك تختار Nest فلترًا يطابق النوع والربط. تُستدعى `catch` مرة. `response.status().json()` ينهي الرد عبر Express. إن كان المعترض قد لفّ التيار، الخطأ وصل إلى الطبقة لأن لا `catchError` حوّله إلى قيمة.

`try/catch` في المتحكم يمنع الوصول إلى هنا. رمي Middleware لا يصل إلى فلتر مربوط على المتحكم. يصل إلى نسخة عامة من الفلتر إن سجّلتها عامًا.

كتابة الرد يدويًا تتجاوز شكل المدمج. إن نسيت `status` يبقى `200` مع جسم خطأ. اضبط الرمز دائمًا من `getStatus()` للاستثناءات المعروفة.

## مثال عملي

عقد الشباك يريد `path` في كل خطأ HTTP. `TicketHttpFilter` على `TicketsController` يضيف المسار لـ `404` و `409` معًا لأنهما `HttpException`. صحة التطبيق بلا هذا الفلتر تبقى على الشكل الافتراضي. اختبار يرسل مقعدًا مبيعًا ويتوقع `statusCode: 409` و `path` ينتهي بمسار الحجز، ولا يتوقع `stack`.

## أخطاء شائعة

- تثبيت `response.status(500)` لكل `HttpException` فيضيع `400`.
- نسيان إنهاء الرد فيبقى العميل معلقًا.
- قراءة `host` وتخزين `response` على حقل الصنف بين الطلبات.
- إعادة رمي بعد `json` أملًا أن فلترًا عامًا يضيف حقلًا. السلسلة لا تُمرَّر.

## متى نستخدمه؟

حين يكون شكل الخطأ جزءًا من العقد ولا يغطي الجسم الافتراضي الحقول التي تريد، مثل `path` أو تسجيل `cause` قبل الكتابة.

## متى لا نستخدمه؟

لترجمة مقعد مبيع. ذلك `SeatTakenException` والجسم عليه. الفلتر للشكل المشترك بين الأخطاء لا لكل قاعدة أعمال. ولا تكتب فلترًا حتى تبتلع `Error` وتخفي `500` المشروع.

## مقارنة

| | الفلتر المدمج | `TicketHttpFilter` |
| --- | --- | --- |
| الجسم | `statusCode` و `message` غالبًا | ما تكتبه في `json` |
| الرمز | من الاستثناء | يجب أن تنسخه |
| فلتر تالٍ | لا يُستدعى إن مسك هذا | نفسه |
| الأعمال | لا يعرف المقعد | لا يجب أن يعرف المقعد |

## تمارين

### تمرين سهل

ما اسما وسيطَي `catch`؟

<details>
<summary>الحل</summary>

الاستثناء، و `ArgumentsHost`.

</details>

### تمرين متوسط

من أين تقرأ `url` ومن أين تقرأ الرمز؟

<details>
<summary>الحل</summary>

`url` من `switchToHttp().getRequest()`. الرمز من `exception.getStatus()` حين يكون الاستثناء `HttpException`.

</details>

### تمرين متقدم

لماذا لا يضيف فلتر عام حقل `requestId` بعد أن كتب فلتر المسار JSON؟

<details>
<summary>الحل</summary>

أول فلتر يمسك الاستثناء ينهي المعالجة. الاستثناء لا يُمرَّر. إن احتجت الحقل فضعه في الفلتر الذي يكتب، أو اجعل فلتر المسار يرث فلترًا أساسيًا يضيف الحقل قبل الإرسال.

</details>

## أسئلة مراجعة

- ماذا تطبّق الواجهة؟
- كيف تصل إلى كائن الرد؟
- لماذا تثبيت `500` خطير؟
- هل يوجد `next` بين الفلاتر؟

## الخلاصة

الفلتر يكتب الرد مرة من الاستثناء والمضيف. الرمز يأتي من `HttpException` لا من ثابت. لا فلتر تالٍ بعد الإمساك. قاعدة المقعد تبقى في الاستثناء، والفلتر يشكّل العقد المشترك.

## ماذا بعد؟

الدرس التالي `@Catch`: كيف تحدد الأنواع، ولماذا القوس الفارغ يمسك `400` مع `500`.

## المصادر الرسمية

- [Exception filters](https://docs.nestjs.com/exception-filters)
