# تمارين الفصل

## سهل

أي حقل يبني صنفًا، وأي حقل يخزن مرجعًا جاهزًا، وأي حقل يشير إلى رمز موجود؟

<details>
<summary>الحل</summary>

`useClass` يبني، `useValue` يخزن المرجع، `useExisting` يشير إلى رمز مسجّل أصلًا.

</details>

## متوسط

`PaymentGateway` مجرد و `StripeGateway` تنفيذ، ومسار قديم يحقن `'PAYMENTS'`. كيف تبقى نسخة Stripe واحدة؟

<details>
<summary>الحل</summary>

سجّل `StripeGateway`، ثم `{ provide: PaymentGateway, useExisting: StripeGateway }` و `{ provide: 'PAYMENTS', useExisting: StripeGateway }`. لا تكرر `useClass` تحت كل رمز.

</details>

## متقدم

عميل بحث يُفتح بـ `await` ويُغلق عند `SIGTERM`. صمّم الرمز والمصنع والإغلاق، وكيف يتخطى الاختبار الشبكة.

<details>
<summary>الحل</summary>

`Symbol` مثل `SEARCH`. `useFactory` غير متزامن يعمل `ping` أو `open` ويرجع صنف `SearchClient` يطبّق `OnModuleDestroy`. التفعيل `enableShutdownHooks`. الاختبار `{ provide: SEARCH, useValue: fake }` فلا يُستدعى المصنع.

</details>
