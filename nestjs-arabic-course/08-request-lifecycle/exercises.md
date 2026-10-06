# تمارين الفصل

أجب من غير فتح الدروس، ثم افتح الحل.

## سهل

1. ما ترتيب Guards العام و Controller والمسار؟
2. هل Exception Filter خطوة على `GET` يعيد قائمة من غير رمي؟

<details>
<summary>الحل</summary>

1. العام، ثم Controller، ثم المسار. داخل المستوى ترتيب الربط.
2. لا. المرشح لاستثناء غير ممسوك. النجاح يمر بخروج Interceptors ثم الاستجابة.

</details>

## متوسط

رتّب طلب `PATCH /loans/9` فيه: `ParseIntPipe` على `:id`، `MemberGuard` على الصنف، `TimingInterceptor` عام، `stamp` على وحدة `@Global()`، `requestId` في `app.use`، و `LoansService.update` أعادت كائنًا.

<details>
<summary>الحل</summary>

1. `requestId` ثم `stamp` (وأي Middleware جذر أو وحدة أبعد حسب الرسم، بعد `@Global()`).
2. `MemberGuard` بعد أي `APP_GUARD`.
3. دخول `TimingInterceptor`.
4. `ParseIntPipe` ضمن مرحلة Pipes، بعد Pipes العام و Controller والمسار إن وُجدت، لأنها Pipe معامل.
5. المعالج ثم `update`.
6. خروج `TimingInterceptor` ثم الاستجابة JSON برمز 200.
7. لا مرشح.

</details>

## متقدم

ثلاثة أعطال على التطبيق نفسه. حدد أول مرحلة تُتخطى وما الذي يرد:

1. `app.use` يرمي قبل `next`.
2. `MemberGuard` يعيد `false` و `@UsePipes(RouteSpecificPipe)` على `PATCH`.
3. الخدمة ترمي، `TimingInterceptor` في `catchError` يرجع `{ ok: false }`، و `LoanPatchFilter` على الدالة.

<details>
<summary>الحل</summary>

1. Guards وما بعدها لا تبدأ. المرشح العام فقط يستطيع صياغة الخطأ. مرشح `PATCH` غير متاح.
2. `RouteSpecificPipe` والمعالج لا يبدآن. استثناء الرفض يُفحص من مرشح المسار إلى Controller ثم العام.
3. طريق النجاح لا يحمل نتيجة `update`. `catchError` أمسك الخطأ، فـ `LoanPatchFilter` لا يرى الاستثناء. الأغلفة الأوسع من المؤقت ترى `{ ok: false }` كقيمة، ثم ترسلها Nest كنجاح.

</details>
