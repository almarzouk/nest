# تمارين الفصل

شباك التذاكر. أجب ثم افتح الحل.

## سهل

1. أي مصنع يبني زينة معامل؟
2. أي دالة على `Reflector` تغلب زينة الدالة على زينة الصنف؟
3. ماذا تكتب فوق الصحة حين تكون الهوية `APP_GUARD`؟

<details>
<summary>الحل</summary>

1. `createParamDecorator`.
2. `getAllAndOverride` مع الدالة ثم الصنف.
3. `@Public(true)` وفرع في الحارس يرجع `true` قبل فحص الترويسة.

</details>

## متوسط

اربط `POST /tickets/refund` و `GET /health`:

- الاسترداد يحتاج مستخدمًا ملصوقًا ودور `refund` وإلا `403` بعد الهوية.
- الدالة تستلم الاسم فقط.
- الصحة بلا ترويسة ترجع `{ ok: true }` رغم الحارس العام.

لا تذكر `exclude`.

<details>
<summary>الحل</summary>

`IdentifyDeskGuard` عام يلصق `{ name, roles }` أو يرمي `401`، ويبدأ بقراءة `Public`. `health` عليها `@Public(true)`. `RolesGuard` بعد الهوية يقرأ `@Roles(['refund'])` عبر `getAllAndOverride`. `refund` عليها تلك الزينة و `@CurrentUser('name')`. من لا يحمل الدور يرجع `false`.

</details>

## متقدم

راجع أربعة أعطال:

1. `@Public(true)` على الصحة والرد `401`.
2. `@Roles(['refund'])` والمستخدم `lead` يُمنع لأن `user.roles` فارغة.
3. الصنف `@Roles(['audit'])` والدالة `@Roles(['refund'])` والحارس يقارن `audit`.
4. `@CurrentUser()` على `health` بعد فتحها ترجع `undefined`.

<details>
<summary>الحل</summary>

1. الحارس لا يقرأ `Public` قبل الترويسة.
2. حارس الهوية يلصق الاسم فقط. أضف `roles` عند اللصق، ورتّب هذا الحارس قبل `RolesGuard`.
3. ترتيب `getAllAndOverride` يضع الصنف أولًا. اجعل `[handler, class]`.
4. مسار `@Public` لا يلصق `user`. لا تضع `@CurrentUser` على الصحة، أو اقبل أن الوسيط غائب لأن الزينة تقرأ ولا تُنشئ مستخدمًا.

</details>
