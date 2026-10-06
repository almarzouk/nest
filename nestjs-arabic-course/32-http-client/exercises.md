# تمارين الفصل

## سهل

1. من أين يُستورد `HttpModule` في هذا الفصل؟
2. أين عنوان الكتاب في رد `HttpService`؟
3. ماذا يعيد `lookup` عند `404` الناشر في العقد الذي كتبناه؟

<details>
<summary>الحل</summary>

1. من `@nestjs/axios`.
2. في `response.data.title`.
3. `null`، لا `NotFoundException` على كتاب المتجر.

</details>

## متوسط

صمم خريطة أخطاء لنداء ISBN: مهلة، غياب، حمولة بلا عنوان، ومفتاح مرفوض. اذكر الرمز الوارد الذي يراه الزائر.

<details>
<summary>الحل</summary>

- مهلة `ECONNABORTED`: `GatewayTimeoutException` أي `504`.
- `404`: `null` حتى تقرر خدمة الكتب العنوان اليدوي.
- `200` بلا `title`: `BadGatewayException`.
- `401` من الناشر: `BadGatewayException` أيضًا، لأن المفتاح سر الخادم لا جلسة الزائر.
- جسم الناشر لا يُنسخ إلى الرسالة.

</details>

## متقدم

الإنتاج يستعمل `@nestjs/axios` والتوثيق الذي فتحه زميلك يعرض `HttpClient` و `baseUrl`. ماذا تثبت في مراجعة الشيفرة حتى لا يختلط المساران في الملف نفسه؟

<details>
<summary>الحل</summary>

- هذا الفصل يبقى `HttpModule.register` و `HttpService` و `baseURL` و `Observable` ثم `firstValueFrom`.
- `@nestjs/http-client` عميل `fetch` بوعود وحقل `baseUrl` ومهلة قد تُكتب مدة نصية. لا تخلط الحقلين في مصنع واحد.
- كلاهما رسمي. اختيار المتجر هنا Axios لأن سلسلة RxJS السابقة تخدمه.
- الاختبار يستبدل `IsbnService` في الحالتين، فلا يتسرب اختيار الحزمة إلى `BooksController`.

</details>
