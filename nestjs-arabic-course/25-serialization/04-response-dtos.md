# Response DTOs

## الهدف من هذا الدرس

فصل صنف الاستجابة عن نموذج الجدول حتى يكون عقد الخروج صريحًا في Nest وفي OpenAPI.

## المشكلة التي يحلها

كيان واحد `User` يُستخدم في `create` وفي الاستجابة. مزخرفات التحقق تطلب `password`، ومزخرفات الإخراج تخفي `passwordHash`، وإضافة Swagger على ملف `.entity.ts` تنشر الاثنين. العميل يرى حقلًا لا يرسله في `GET` وحقلًا لا يقبله في `POST`. التعديل على الكيان يكسر اتجاهًا.

## الفكرة ببساطة

`CreateUserDto` للداخل. `UserResponse` للخارج. الخدمة تقبل صف Prisma وتعيد `UserResponse` بحقول مختارة. الاعتراض والمخطط يطبَّقان على صنف الخروج. وثيقة `@ApiOkResponse({ type: UserResponse })` تشير إليه. الجدول يبقى أوسع من العقد.

## التعريف التقني

```ts
export class UserResponse {
  @ApiProperty()
  id: number;

  @ApiProperty({ example: "ada@library.test" })
  email: string;

  @ApiProperty({ enum: ["reader", "editor"] })
  role: string;

  constructor(partial: Partial<UserResponse>) {
    Object.assign(this, partial);
  }
}
```

```ts
toResponse(row: { id: number; email: string; role: string }): UserResponse {
  return new UserResponse({ id: row.id, email: row.email, role: row.role });
}
```

لا معامل `passwordHash` في `toResponse`. TypeScript يمنع النسخ العرضي إن لم تضف الحقل إلى الصنف. هذا أدق من `@Exclude` وحده على كيان مشترك، ويمكن الجمع: الصنف الضيق زائد اعتراض إن رغبت بطبقة ثانية.

`PostResponse` يحمل `id` و `title` و `published` و `authorId`. `CommentResponse` يحمل `id` و `body` و `postId`. لا كائن `author` كاملًا.

مع `@SerializeOptions({ type: UserResponse })` يستطيع المعالج إرجاع الكائن الضيق العادي. إن أعاد الصف الواسع، التحويل إلى `UserResponse` عبر `plainToInstance` قد ينسخ الخصائص الزائدة إلى النسخة إن سمح `excludeExtraneousValues` بخلاف ذلك. في `class-transformer`، `@Exclude` على الصنف مع `@Expose` على الحقول، أو خيار `excludeExtraneousValues: true`، يسقط الزائد. صفحة Nest تمرّر خيارات `@SerializeOptions` إلى `instanceToPlain`. لضبط التحويل من كائن عادي، المسار الموثّق هو `type`. أبقِ `toResponse` ينسخ ثلاثة حقول فقط حتى لا تعتمد على خيار تنساه.

مخطط Zod `userResponseSchema` قائمة سماح مكافئة. `StandardSchemaSerializerInterceptor` مع `@SerializeOptions({ schema })` يحذف `passwordHash` حتى من صف كامل. مصفوفة المستخدمين: مرّر مخطط العنصر. عدم تطابق النوع `500`. الدورة تستخدم الأصناف على المقالات. المخطط خيار حين يكون التحقق أصلًا بـ Zod.

`@ApiHideProperty` على كيان مشترك يخفي الوثيقة فقط. `UserResponse` يلغي الحاجة إليه لأن الحقل غير موجود.

## Mental Model

```text
صف Prisma واسع
  toResponse يختار
  UserResponse هو JSON والعقد
CreateUserDto لا يُعاد في GET
```

## أبسط مثال

```ts
@Get(":id")
@ApiOkResponse({ type: UserResponse })
@SerializeOptions({ type: UserResponse })
findOne(@Param("id") id: string) {
  return this.users.findOne(id);
}
```

## شرح المثال

الخدمة تعيد كائنًا يطابق `UserResponse`. الخيار `type` يحمي إن تسرب حقل إضافي عليه مزخرف استبعاد. `@ApiOkResponse` يربط الوثيقة بالصنف نفسه. اختبار العقد يقارن مفاتيح JSON بمفاتيح الصنف.

لا تعيد `CreateUserDto` من `GET`. ذلك الصنف فيه `password` وقد يخلو من `id`.

## ماذا يحدث داخليًا؟

المعالج يرجّع القيمة. الاعتراض يطبّق `type` إن وُجد ثم `instanceToPlain`. Swagger يقرأ الصنف عند بناء الوثيقة لا عند الطلب. لذلك يمكن أن تصدق الوثيقة ويكذب JSON إن رجعت كيانًا آخر بلا اعتراض. الاختبار الذي يطلب المسار يثبت الاثنين معًا.

تداخل `posts` داخل مستخدم يعيد مصفوفة مقالات. إن كان النوع `UserResponse` بلا علاقة، لا تضمّن `posts` في النسخ. نقطة `GET /users/:id/posts` تعيد `PostResponse[]` بعقدها.

## مثال عملي

التسجيل يعيد `UserResponse` بـ `201` من غير كلمة ومن غير تجزئة. `GET /me` يعيد الصنف نفسه. `PATCH` للبريد يعيد الصنف بعد التحديث. ثلاث استجابات ودالة `toResponse` واحدة. فرق OpenAPI: الإنشاء `@ApiCreatedResponse` والقراءة `@ApiOkResponse`، والنوع واحد.

علّق اختبارًا يفشل إن وُجد المفتاح `password` أو `passwordHash`.

## أخطاء شائعة

- كيان واحد للداخل والخارج.
- `toResponse` بنسخ `{ ...row }`.
- توثيق `User` المولَّد من Prisma كـ `type` في Swagger.
- مخطط استجابة يصف المصفوفة بينما الاعتراض ينتظر مخطط العنصر.

## متى نستخدمه؟

لكل نموذج يغادر API نحو عميل، حتى لو تطابق مع الجدول اليوم.

## متى لا نستخدمه؟

داخل معاملة Prisma بين خدمات. الخدمة الداخلية قد تحتاج التجزئة لـ `verify`. الحد على المعالج الذي يواجه الشبكة.

## مقارنة

| `UserResponse` | كيان الجدول |
| --- | --- |
| حقول العقد | أعمدة التخزين |
| بلا `passwordHash` | العمود موجود |
| `type` في OpenAPI | إضافة CLI قد تنشر الأسرار |

## تمارين

### تمرين سهل

أي صنف توثّقه في `@ApiOkResponse` لـ `GET /me`؟

<details>
<summary>الحل</summary>

`UserResponse`، لا `CreateUserDto` ولا كيان الجدول.

</details>

### تمرين متوسط

ماذا ينسخ `toResponse` من صف المستخدم؟

<details>
<summary>الحل</summary>

`id` و `email` و `role`. التجزئة لا تُمرَّر إلى البناء.

</details>

### تمرين متقدم

لماذا `{ ...row }` داخل `new UserResponse` يعيد الخطر؟

<details>
<summary>الحل</summary>

ينسخ `passwordHash` إلى النسخة. من غير `@Exclude` و اعتراض يخرج في JSON. النسخ المسمى للحقول الثلاثة يمنع ذلك في النوع.

</details>

## أسئلة مراجعة

- لماذا يُفصل DTO الدخول عن صنف الخروج؟
- ماذا يحدث إن خالف الصف مخطط Zod للاستجابة؟
- هل `@ApiHideProperty` يحذف الحقل من الشبكة؟

## الخلاصة

صنف استجابة ضيق هو العقد. `toResponse` يختار الحقول. الوثيقة تشير إلى الصنف نفسه الذي يسلسله الاعتراض.

## ماذا بعد؟

[ما لا يجوز أن يخرج](./05-what-must-never-leak.md) قائمة أسرار المكتبة.

## المصادر الرسمية

- NestJS — Serialization: https://docs.nestjs.com/application/serialization
- NestJS — OpenAPI types: https://docs.nestjs.com/openapi/types-and-parameters
