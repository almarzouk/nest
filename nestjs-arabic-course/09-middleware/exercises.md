# تمارين الفصل

أجب من غير فتح الدروس. المجال مكتب إعارة: `LoansController` و `CopiesController` و `GET /health`.

## سهل

1. ما المعاملات الثلاثة لدالة Middleware على محوّل Express؟
2. أي سطر تسجيل يعمل قبل `configure` في أي وحدة؟
3. ماذا تفعل إن لم ترد إنهاء الرد داخل الدالة؟

<details>
<summary>الحل</summary>

1. `req` و `res` و `next`.
2. `app.use(...)` في `main.ts`.
3. تنادي `next()`.

</details>

## متوسط

ارسم ربطًا واحدًا:

- `requestId` على كل الطلبات بلا حقن.
- `CopyClockMiddleware` على `CopiesController` فقط، والساعة مزوّد.
- `traceLoan` على `LoansController` ما عدا `POST /loans`.

اذكر الملف الذي يعيش فيه كل سطر.

<details>
<summary>الحل</summary>

- `main.ts`: `app.use(requestId)` بعد `NestFactory.create`.
- `copies.module.ts`: `providers: [CopyClock]`، و `configure` فيها `consumer.apply(CopyClockMiddleware).forRoutes(CopiesController)`.
- `loans.module.ts`: `consumer.apply(traceLoan).exclude({ path: 'loans', method: RequestMethod.POST }).forRoutes(LoansController)`.
- ترتيب الطلب على `GET /copies`: `requestId` ثم `CopyClockMiddleware`. `traceLoan` لا يعمل لأن المسار ليس على `LoansController`.

</details>

## متقدم

ظهر العطل التالي في المراجعة. لكل بند اذكر الطبقة الصحيحة والسبب المختصر.

1. فحص `roles` المخزّنة على `DELETE /loans/:id` داخل `use`.
2. `express.json()` مرة ثانية مع الإعداد الافتراضي لـ `NestFactory.create`.
3. فلتر `@Catch()` على `LoansController` لا يرى `UnauthorizedException` رُمي من Middleware المكتب.
4. مسجّل في وحدة `@Global()` يطبع `x-request-id` فارغًا، والمولّد يعيش في `LoansModule`.
5. `GET /health` يرجع `403` بعد `APP_GUARD`، والفريق أضاف `exclude('health')` على Middleware.

<details>
<summary>الحل</summary>

1. Guard مع `ExecutionContext`. Middleware لا يعرف المعالج التالي.
2. المحوّل سجّل المحلّل. التخصيص يبدأ بـ `bodyParser: false` ثم تسجيل واعٍ، لا بمصنع ثانٍ فوق الافتراضي.
3. استثناء Middleware تراه الفلاتر العامة فقط (`useGlobalFilters` أو `APP_FILTER`). فلتر المتحكم لا يُختار قبل تحديد المسار.
4. مولّد المعرّف يجب أن يسبق القارئ. انقله إلى `app.use` أو إلى وحدة أقرب إلى الجذر من المسجّل العام. `@Global()` يقدّم Middleware تلك الوحدة على الجذر وعلى `LoansModule`.
5. `exclude` يطرح من سلسلة Middleware. الحارس العام يعمل بعد ذلك. فتح الصحة يتم بعلامة على المسار يقرأها الحارس، مثل `@Public` في فصل الزينة، لا بثقب في Middleware.

</details>

## سؤال إضافي قبل الإغلاق

`ShelfClockMiddleware` يرمي `UnauthorizedException` عند غياب ترويسة. فلتر مكتوب على `LoansController` لا يغيّر جسم الرد. اكتب أين تسجّل الفلتر الذي يستطيع ذلك، ولماذا `@UseFilters` على صنف Middleware لا يكفي.

<details>
<summary>الحل</summary>

الفلتر العام عبر `app.useGlobalFilters()` أو مزوّد `APP_FILTER`. الصفحة تقول إن استثناء Middleware، لأن المعالج لم يُختر بعد، تراه الفلاتر العامة فقط، وإن ربط `@UseFilters()` على صنف Middleware غير مدعوم.

</details>
