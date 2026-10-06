# تمارين الفصل

## سهل

ما الحقل الذي لا يُحذف من كائن `DynamicModule`؟

<details>
<summary>الحل</summary>

`module`، ويشير إلى صنف الوحدة. بقية الحقول اختيارية وتُدمَج مع `@Module`.

</details>

## متوسط

`USER_ID_PREFIX` في الإعداد و `ORDER_ID_PREFIX` في الإعداد. أي دالة على `IdGeneratorModule`، وأين تعيش كل واحدة؟

<details>
<summary>الحل</summary>

`registerAsync` داخل `UsersModule` بمصنع يقرأ المفتاح الأول، و `registerAsync` داخل `OrdersModule` للمفتاح الثاني. ليست `forRootAsync` واحدة في الجذر لأن الخيار ليس مشتركًا.

</details>

## متقدم

عامل serverless يبني PDF أحيانًا. الوحدة فيها `PdfService` فقط. اكتب خطوات التحميل، واذكر لماذا لا تضع `PdfController` هناك، وأين تغلق ملفًا مؤقتًا.

<details>
<summary>الحل</summary>

`import('./pdf.module.js')` ثم `lazyModuleLoader.load(() => PdfModule)` ثم `moduleRef.get(PdfService)`. المتحكم المتأخر لا يسجل مسارًا بعد الاستماع. خطافات الوحدة الكسولة لا تعمل، فأغلق الملف في الدالة أو في خدمة ساخنة لها `onModuleDestroy`.

</details>

## سؤال إعادة التصدير

لماذا `exports: [DatabaseModule.forRoot(url)]` من `BooksModule` خطأ حتى لو تطابق العنوان؟

<details>
<summary>الحل</summary>

لأنه استدعاء تكوين ثانٍ. بعد النداء الأول في الجذر أو النواة يُعاد تصدير الصنف `DatabaseModule` وحده.

</details>
