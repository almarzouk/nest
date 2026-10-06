# تمارين الفصل

## سهل

1. أي مسار يفتح Swagger UI بعد `setup("api", ...)`؟
2. أي مزخرف يملأ خصائص النموذج قبل إضافة CLI؟
3. أي دالة تسجّل Bearer في الباني؟

<details>
<summary>الحل</summary>

1. `/api`.
2. `@ApiProperty`.
3. `addBearerAuth()`.

</details>

## متوسط

1. لماذا يُغلَّف `createDocument` في دالة؟
2. من أين يُستورد `PartialType` مع الإضافة؟
3. ماذا يلزم Fastify زيادة على `@nestjs/swagger`؟

<details>
<summary>الحل</summary>

1. لتأخير البناء حتى الطلب، ولتجنّب فقدان بادئة الإصدار عند البناء المبكر.
2. من `@nestjs/swagger`.
3. `@fastify/static`.

</details>

## متقدم

`GET /posts` عام ويعيد غلاف صفحة. `POST /posts` يحتاج جلسة أو Bearer ويعيد مقالًا بلا تجزئة. `POST /users` يرفض `role`.

1. أي عمليات تحمل قفلًا؟
2. كيف يُوثَّق الغلاف حتى يحصل المولد على اسم نوع؟
3. ماذا تفعل إضافة CLI بكيان فيه `passwordHash` إن مرّرته كـ `type`؟

<details>
<summary>الحل</summary>

1. الإنشاء وما شابهه. القراءة العامة والتسجيل بلا قفل.
2. `allOf` مع `getSchemaPath` و `title` مثل `PaginatedResponseOfPostResponseDto`، مع `@ApiExtraModels`.
3. قد تنشر الحقل لأن الملف `.entity.ts` يُحلَّل. استخدم `PostResponseDto` أو `@ApiHideProperty`.

</details>
