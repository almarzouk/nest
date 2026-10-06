# تمارين الفصل

## سهل

1. أي اعتراض يطبّق `@Exclude`؟
2. من أي حزمة يأتي `@Exclude`؟
3. هل `@ApiHideProperty` يمنع الحقل في JSON؟

<details>
<summary>الحل</summary>

1. `ClassSerializerInterceptor`.
2. `class-transformer`.
3. لا. يخفي الوثيقة فقط.

</details>

## متوسط

1. لماذا لا يعمل `@Exclude` على كائن Prisma العادي؟
2. ماذا ينسخ `toResponse`؟
3. أي رمز حين تفشل قيمة الاستجابة أمام مخطط Zod؟

<details>
<summary>الحل</summary>

1. `instanceToPlain` يعمل على نسخة الصنف. استخدم `new` أو `@SerializeOptions({ type })`.
2. `id` و `email` و `role` فقط.
3. `500` لأن العيب من الخادم.

</details>

## متقدم

عمود `passwordHash` ما زال في الجدول. `GET /me` يعيد نتيجة `validate`. وثيقة OpenAPI تشير إلى `UserResponse` بلا السر. السجل يطبع المستخدم عند `409`.

1. لماذا يرى العميل التجزئة رغم الوثيقة؟
2. ماذا تغيّر في المعالج؟
3. ماذا يبقى في رسالة التعارض؟

<details>
<summary>الحل</summary>

1. الوثيقة لا تسلسل الاستجابة، والكائن المعاد واسع.
2. `select` أو `toResponse` إلى `UserResponse`، ولا تُعد كائن التحقق الخام.
3. نص ثابت بلا صف وبلا `meta`. السجل يأخذ `userId` فقط.

</details>
