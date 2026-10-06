# تمارين الفصل

أجب على متجر يخدم JSON تحت `/api` وواجهة مبنية في `client/dist`.

## سهل

1. أي خيار يحدد مجلد القرص، وأي خيار يحدد بادئة العنوان؟
2. هل غلاف الكتاب الخاص ملف ثابت؟

<details>
<summary>الحل</summary>

1. `rootPath` المجلد، و `serveRoot` البادئة.
2. لا. هو تنزيل بعد ترجمة معرف الكتاب إلى مفتاح، عبر `StreamableFile`.

</details>

## متوسط

اكتب كائن `forRoot` لواجهة على جذر الموقع و API تحت `/api`. اذكر طلبين تستخدمهما للتحقق.

<details>
<summary>الحل</summary>

```ts
ServeStaticModule.forRoot({
  rootPath: join(process.cwd(), 'client', 'dist'),
  exclude: ['/api/{*path}'],
});
```

`GET /` يعيد HTML. `GET /api/missing` يعيد `404` وجسمه ليس `index.html`. على Fastify استبدل `exclude` بـ `renderPath` منتظم.

</details>

## متقدم

نشرت الواجهة على CDN و Nest تستقبل `/api` فقط، لكن `index.html` ما زال يُخدم من Nest وبـ `Cache-Control` لسنة على `index.html` وعلى الحزم. ماذا تكسر، وما الذي تحذفه من `AppModule`؟

<details>
<summary>الحل</summary>

Nest لم تعد مصدر الواجهة، فوجود `ServeStaticModule` يكرر الأصل ويبقي خطر البدل على مسارات API إن ضعف الاستثناء. احذف تسجيل الوحدة من عملية الـ API. تخزين `index.html` لسنة يعلّق المتصفح على أسماء حزم قديمة بعد النشر. الحزم المجزّأة هي التي تطيق عمرًا طويلًا، والفهرس يُعاد التحقق منه عند الـ CDN لا عند Nest.

</details>
