# HttpService

## الهدف من هذا الدرس

استدعاء الناشر عبر `HttpService` وتحويل أول قيمة في التيار إلى شكل المجال.

## المشكلة التي يحلها

بعد التسجيل، الحقن بلا استعمال يترك ISBN فارغًا. استدعاء `this.http.get()` بلا اشتراك لا يرسل شيئًا، لأن التيار بارد كما في الفصل السابق. وإرجاع `Observable` من `IsbnService` ينشر RxJS في `BooksService` التي تعمل بـ `async`.

## الفكرة ببساطة

`HttpService` من `@nestjs/axios` يعرض `get` و `post` وغيرهما. كل تابع يعيد `Observable` لاستجابة Axios. خذ أول قيمة بـ `firstValueFrom` داخل خدمة الغلاف، وأعد `Promise` بالعنوان فقط. `pipe` قبل `firstValueFrom` مكان الترجمة والخطأ، وتفصيل الخطأ في الدرس التالي.

## التعريف التقني

```ts
import { Injectable } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

interface IsbnPayload {
  title: string;
}

@Injectable()
export class IsbnService {
  constructor(private readonly http: HttpService) {}

  async lookup(isbn: string): Promise<{ title: string }> {
    const response = await firstValueFrom(
      this.http.get<IsbnPayload>(`/isbns/${encodeURIComponent(isbn)}`),
    );
    return { title: response.data.title };
  }
}
```

`get<IsbnPayload>` يصف `data` لا يضمنها وقت التشغيل. إن غاب `title` ارمِ خطأ ترجمة بدل إرجاع `{ title: undefined }`. المسار نسبي لأن `baseURL` على الوحدة. `encodeURIComponent` يمنع حقن شرطة مائلة في المسار.

`response.data` جسم المنبع. `status` و `headers` على الكائن نفسه. لا تمرر `response` إلى المتحكم.

## Mental Model

```text
lookup(isbn)
  http.get تيار بارد
    firstValueFrom يشترك مرة
      Axios يرسل
        data.title -> { title }
```

## أبسط مثال

استعمال من خدمة الكتب:

```ts
async create(input: { title?: string; isbn?: string }) {
  const enriched = input.isbn ? await this.isbn.lookup(input.isbn) : null;
  return this.books.insert({
    title: input.title ?? enriched?.title ?? 'Untitled',
  });
}
```

هذا الشكل ينتظر نجاح `lookup`. إن كان الإثراء اختياريًا عند العطل، فالتقاط الخطأ هنا أو داخل `lookup` قرار الدرس التالي. المهم أن `create` لا ترى `Observable`.

## شرح المثال

اشتراك واحد عبر `firstValueFrom`. لا `subscribe` يدوي قبل `await`. التابع `get` لا يفتح الشبكة عند استدعائه حتى يشترك `firstValueFrom`. إن نسخت التيار إلى متغير واشتركت للاختبار ثم استدعيت `firstValueFrom` على تيار بارد جديد، فهما طلبان. ابنِ التيار داخل الدالة مرة واحدة.

## ماذا يحدث داخليًا؟

Axios ينفّذ الطلب عند الاشتراك ويعيد الاستجابة ثم يكتمل التيار. `firstValueFrom` يأخذ `data` المغلفة في الاستجابة ويلغي الاشتراك بعدها. جسم JSON يُحلَّل في Axios حسب `Content-Type`. خطأ شبكة أو رمز حالة يرفضه Axios افتراضيًا (رموز خارج 2xx) فيرفض الوعد. لذلك `lookup` كما هو مكتوب يرمي عند `404` الناشر. ترجمته في `pipe` قبل `firstValueFrom` أو في `catch` حول `await`. الدرس التالي يختار شكلًا واحدًا.

## مثال عملي

حقل ناقص من الناشر:

```ts
if (typeof response.data?.title !== 'string' || response.data.title.length === 0) {
  throw new BadGatewayException('isbn payload missing title');
}
```

هذا فشل عقد لا غياب ISBN. الغياب `404` سيُترجم إلى `null` لاحقًا إن قررت ذلك. الحمولة الناقصة مع `200` تعني أن الناشر غيّر الوثيقة، و `502` أصدق من عنوان فارغ في الجدول.

## أخطاء شائعة

- نسيان `firstValueFrom` وإرجاع التيار إلى متحكم يعامل كأنه كائن كتاب.
- قراءة `response.title` بدل `response.data.title`.
- بناء المسار بتركيب `isbn` بلا ترميز.
- اشتراك إضافي للتسجيل يطلق طلبًا ثانيًا.

## متى نستخدمه؟

داخل خدمة المنبع بعد `HttpModule.register`. واجهة الدالة العامة `Promise`.

## متى لا نستخدمه؟

في المعترض العام لكل الطلبات الواردة. `HttpService` للصادر. الوارد يبقى متحكمًا وخدمة مجال.

## مقارنة

| الاستدعاء | الشبكة |
| --- | --- |
| `this.http.get(url)` بلا اشتراك | لا تُرسل |
| `firstValueFrom(this.http.get(url))` | مرة واحدة |
| `subscribe` ثم `firstValueFrom` على التيار البارد نفسه بشكل منفصل | مرتان إن أعدت بناء الوصفة أو إن كان الأثر في المصنع |

## تمارين

### تمرين سهل

أين يقع عنوان الكتاب في استجابة `HttpService.get`؟

<details>
<summary>الحل</summary>

في `response.data.title` لا على جذر الاستجابة.

</details>

### تمرين متوسط

لماذا `lookup` تُعلن `Promise` لا `Observable`؟

<details>
<summary>الحل</summary>

حتى تبقى `BooksService` على `async` ولا تنتشر معاملات RxJS في المجال.

</details>

### تمرين متقدم

`200` بلا خاصية `title`. أي استثناء وارد، ولماذا ليس `null`؟

<details>
<summary>الحل</summary>

`BadGatewayException` لأن العقد كُسر. `null` محجوز لغياب ISBN إن اخترت ترجمة `404` الخارجي إلى «غير موجود عند الناشر».

</details>

## أسئلة مراجعة

- أي دالة تشغّل التيار البارد هنا؟
- لماذا `encodeURIComponent`؟
- ماذا لا يُعاد إلى `BooksController`؟

## الخلاصة

`HttpService.get` تيار. `firstValueFrom` عند حافة `IsbnService` يعيد `{ title }`. شكل الناشر لا يتجاوز هذه الخدمة.

## ماذا بعد؟

ترجمة `AxiosError` والمهلة إلى استثناءات Nest.

## المصادر الرسمية

- NestJS — HTTP module (Axios): https://docs.nestjs.com/techniques/http-module
- RxJS — `firstValueFrom`: https://rxjs.dev/api/index/function/firstValueFrom
