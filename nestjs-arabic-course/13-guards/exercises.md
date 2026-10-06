# تمارين الفصل

شباك التذاكر. أجب ثم افتح الحل.

## سهل

1. ماذا يفعل الإطار حين ترجع `canActivate` القيمة `false`؟
2. أي مستوى ربط يغلق `GET /health` حتى لو لم تزين متحكم الصحة؟
3. من يعمل أولًا: حارس على الصنف أم حارس على الدالة؟

<details>
<summary>الحل</summary>

1. يرمي `ForbiddenException` والرد `403`.
2. الربط العام: `APP_GUARD` أو `useGlobalGuards`.
3. حارس الصنف، بعد الحراس العامين وقبل حارس الدالة.

</details>

## متوسط

صمم ربط `POST /tickets` و `POST /tickets/refund` و `GET /health`:

- كل مسارات التذاكر تحتاج `x-desk-name` وإلا `401`.
- الاسترداد يحتاج الاسم `lead` وإلا `403`.
- الصحة تبقى بلا بوابة.

اذكر الزينة ومستوى كل حارس، لا جسم JWT.

<details>
<summary>الحل</summary>

`IdentifyDeskGuard` على `TicketsController` يرمي `UnauthorizedException` عند غياب الترويسة ويلصق `user`. `RefundGuard` على دالة `refund` فقط يرجع `false` لغير `lead`. لا `APP_GUARD`. `HealthController` بلا `@UseGuards`. الترتيب على الاسترداد: الهوية ثم `RefundGuard`.

</details>

## متقدم

بعد النقل إلى `APP_GUARD` ظهر ما يلي. فسّر كل سطر:

1. `GET /health` صار `401`.
2. `POST /auth/login` صار `401` قبل قراءة الجسم.
3. `exclude({ path: 'health', method: RequestMethod.GET })` لم يفتح الصحة.
4. حارس الدور العام يرى `user` فارغًا رغم أن حارس الهوية عام أيضًا.

<details>
<summary>الحل</summary>

1 و 2. الربط العام يشمل كل الدوال. الدخول والصحة يحتاجان علامة لاحقة مثل `@Public` يقرأها الحارس ويرجع `true`، أو إبقاء الحارس على متحكم التذاكر.
3. `exclude` يطرح من سلسلة Middleware. الحارس يعمل بعد تلك السلسلة.
4. ترتيب تسجيل `APP_GUARD` يقدّم الدور على الهوية. سجّل حارس الهوية أولًا حتى يلصق `user` قبل قراءة الدور.

</details>
