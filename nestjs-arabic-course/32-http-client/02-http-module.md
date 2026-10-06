# HttpModule

## الهدف من هذا الدرس

تسجيل عميل Axios كوحدة ديناميكية حتى يُحقن `HttpService` في خدمة الغلاف فقط.

## المشكلة التي يحلها

`axios.get` مباشرة داخل الخدمة يعمل ويتجاوز اختبار Nest ومهلة موحّدة. كل دالة تنسخ `baseURL` والمفتاح. `HttpModule.register` يبني النسخة مرة ويمنحها لمن يستورد الوحدة.

## الفكرة ببساطة

الحزمة `@nestjs/axios`. `HttpModule.register` خيارات Axios مثل `baseURL` و `timeout` و `maxRedirects`. الوحدة التي تستوردها تستطيع حقن `HttpService`. صفحة التوثيق الحالية لـ «HTTP client» تصف `@nestjs/http-client` و `HttpClientModule.register` بوعود `fetch`. هذا الدرس عن عميل Axios الذي بقي مدعومًا، لا عن ذلك المسار.

## التعريف التقني

```bash
npm i @nestjs/axios axios
```

```ts
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { IsbnService } from './isbn.service.js';

@Module({
  imports: [
    HttpModule.register({
      baseURL: 'https://isbn.example/v1',
      timeout: 800,
      maxRedirects: 2,
    }),
  ],
  providers: [IsbnService],
  exports: [IsbnService],
})
export class IsbnModule {}
```

`timeout` بالميلي ثانية على طلب Axios نفسه، فيُقطع المقبس لا الملاحظ فقط. `baseURL` يُلصق بالمسار النسبي في `get('/books/978...')`.

`registerAsync` عندما يأتي العنوان من `ConfigService`:

```ts
HttpModule.registerAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    baseURL: config.getOrThrow<string>('ISBN_BASE_URL'),
    timeout: config.get<number>('ISBN_TIMEOUT_MS') ?? 800,
    headers: { authorization: `Bearer ${config.getOrThrow<string>('ISBN_TOKEN')}` },
  }),
});
```

لا تضع الرمز في المستودع. الترويسة هنا ترويسة الصادر نحو الناشر.

## Mental Model

```text
IsbnModule
  HttpModule.register -> HttpService واحد لهذه الوحدة
  IsbnService يحقنه
BooksModule تستورد IsbnModule وترى IsbnService المصدَّر فقط
```

## أبسط مثال

الجذر يستورد الميزة لا خيارات Axios:

```ts
@Module({
  imports: [IsbnModule, BooksModule],
})
export class AppModule {}
```

`BooksService` تحقن `IsbnService` ولا تحقن `HttpService`. تغيير الناشر يبقى داخل `IsbnModule`.

## شرح المثال

`exports: [IsbnService]` يمنع الوحدات الأخرى من استخدام Axios العام بلا ترجمة. إن صدّرت `HttpModule` نفسه صار كل متحكم قادرًا على نداء أي عنوان. أبقِ العميل خلف خدمتك. `timeout: 800` يطابق مهلة الوكيل القصيرة من فصل RxJS، على المصدر هذه المرة.

## ماذا يحدث داخليًا؟

`register` ينشئ مزوّدات Axios و `HttpService` في نطاق الوحدة المستورِدة. كل استيراد لـ `HttpModule.register` بكائن مختلف يصنع عميلًا لإعداد تلك الوحدة، وهذا مطلوب إن كان لديك ناشر ISBN وخدمة بريد بعناوين مختلفة. لا تسجّل عميلًا عامًا واحدًا بـ `baseURL` الناشر ثم تستخدمه للبريد. `HttpService.axiosRef` يصل إلى Axios لمعترض مكتبة. القراءة العادية تبقى على التوابع التي تعيد تيارًا.

## مثال عملي

خدمتان صادرتان، وحدتان:

```text
IsbnModule   baseURL الناشر   timeout 800
CoverModule  baseURL الصور    timeout 2000
```

صورة الغلاف أبطأ من بحث النص. مهلة واحدة على عميل عام تقطع الصورة أو تُطيل البحث. `BooksModule` تستورد الاثنتين وتتحدث إلى خدمتين لا إلى `HttpService`.

## أخطاء شائعة

- `HttpModule` من `@nestjs/common`. المسار `@nestjs/axios` لهذا الفصل.
- اعتبار `HttpClientModule` الاسم نفسه. ذاك عميل `fetch` في التوثيق الحالي.
- تصدير `HttpModule` من وحدة مشتركة بلا `baseURL` لكل منبع.
- كتابة `timeout: 800` وظن أنها ثوانٍ.

## متى نستخدمه؟

لكل منبع HTTP خارجي له عنوان ومهلة. وحدة لكل منبع أو لكل عقد.

## متى لا نستخدمه؟

لنداء دوال Nest داخل العملية. ولا تستبدله بـ `fetch` خام في المتحكم إن كنت تريد الحقن والاختبار على `IsbnService`.

## مقارنة

| التسجيل | الحزمة | الإرجاع |
| --- | --- | --- |
| `HttpModule.register` | `@nestjs/axios` | `HttpService` و `Observable` |
| `HttpClientModule.register` | `@nestjs/http-client` | `HttpClient` و `Promise` |

المتجر في هذا الفصل يكمّل الصف الأول.

## تمارين

### تمرين سهل

من أي حزمة يُستورد `HttpModule` هنا؟

<details>
<summary>الحل</summary>
من `@nestjs/axios`.
</details>

### تمرين متوسط

لماذا لا يُحقن `HttpService` في `BooksController`؟

<details>
<summary>الحل</summary>
حتى يبقى عنوان الناشر والمفتاح والترجمة داخل `IsbnService`. المتحكم يعرف الكتب لا Axios.
</details>

### تمرين متقدم

لماذا عميل البريد لا يشارك `baseURL` عميل ISBN؟

<details>
<summary>الحل</summary>
لأن المهلة والعنوان والمفتاح مختلفة. `register` منفصل في كل وحدة يمنع لصق مسار البريد على مضيف الناشر.
</details>

## أسئلة مراجعة

- ما وحدة `timeout` في `register`؟
- ماذا تصدّر `IsbnModule`؟
- أي تسجيل تستعمله عندما يأتي العنوان من الإعداد؟

## الخلاصة

`HttpModule.register` يبني عميل Axios لوحدة المنبع. الخدمة المصدَّرة هي واجهة المجال، والمهلة على المقبس لا على المعترض فقط.

## ماذا بعد؟

`HttpService.get` والتيار، ثم `firstValueFrom` عند حافة الخدمة.

## المصادر الرسمية

- NestJS — HTTP module (Axios): https://docs.nestjs.com/techniques/http-module
- NestJS — HTTP client (`fetch`): https://docs.nestjs.com/application/http-client
