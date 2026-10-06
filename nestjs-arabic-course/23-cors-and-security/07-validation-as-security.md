# Validation كطبقة أمن

## الهدف من هذا الدرس

جعل `ValidationPipe` حدًا يرفض الشكل الزائد والحقول التي لا يملك العميل حق كتابتها، مع معرفة ما لا يفحصه الأنبوب.

## المشكلة التي يحلها

`POST /users` يقبل الجسم كاملًا. عميل يضيف `role: "editor"` أو `passwordHash` فيُحفظان إن نسخ المعالج `dto` إلى `create`. هذا تعيين جماعي: حقل لم تصممه في العقد يغيّر صلاحية أو يكتب تجزئة جاهزة. أنبوب بلا `whitelist` يمرّر الحقول الزائدة إلى الخدمة.

## الفكرة ببساطة

الأنبوب يتحقق من DTO قبل المعالج. `whitelist` يحذف ما لا مزخرف له. `forbidNonWhitelisted` يرفض الطلب بدل الحذف الصامت. الحقول الحساسة لا تُعلن في DTO الإنشاء أصلًا: الدور يحدده الخادم، والتجزئة ناتج `PasswordHasher`. الأنبوب لا يهرّب HTML ولا يربط معاملات SQL ولا يثبت الهوية.

## التعريف التقني

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

من صفحة التحقق. DTO التسجيل:

```ts
import { IsEmail, IsString, MaxLength, MinLength } from "class-validator";

export class RegisterDto {
  @IsEmail()
  @MaxLength(254)
  email: string;

  @IsString()
  @MinLength(10)
  @MaxLength(200)
  password: string;
}
```

لا `role` ولا `passwordHash` ولا `id`. الخدمة:

```ts
const passwordHash = await this.passwordHasher.hash(dto.password);
return this.prisma.user.create({
  data: { email: dto.email, passwordHash, role: "reader" },
});
```

`role: "reader"` ثابت هنا أو من قاعدة دعوة، لا من الجسم. تحديث الدور مسار آخر بصلاحية `editor` أو إدارة، مع سياسة.

`transform: true` يحوّل الاستعلام النصي إلى أرقام حين يوجد `@Type`. من غير ذلك قد تفشل مقارنة `page` أو تُمرَّر كنص إلى Prisma.

الحد الأقصى للطول يحد الذاكرة. `PasswordHasher.hash` يرفض الفارغ وما فوق 4KiB بـ `RangeError`. الحد في DTO أوضح للعميل كـ `400` قبل التجزئة.

فشل الأنبوب `400`. ذلك ليس `401` ولا `403`.

## Mental Model

```text
جسم JSON
  pipe: شكل مسموح فقط
  خدمة: تقرر الدور والتجزئة
  سياسة: هل هذا المستخدم يكتب
  SQL: معاملات منفصلة
```

## أبسط مثال

```ts
@Post("users")
@Public()
register(@Body() dto: RegisterDto) {
  return this.users.register(dto);
}
```

## شرح المثال

`@Public()` يعني أن التسجيل بلا جلسة، لا أن كل حقل مباح. الأنبوب يرفض `{ email, password, role: "editor" }` لأن `role` غير مزخرف و `forbidNonWhitelisted` مفعّل. مع `whitelist` وحده قد يُحذف `role` بصمت ويعود `201`، فيظن العميل أنه صار محررًا. الرفض الصريح أوضح.

`password` لا يُحفظ. الخدمة تجزئ ثم تنسى النص. الاستجابة لا تعيد التجزئة، والتفاصيل في فصل التسلسل.

## ماذا يحدث داخليًا؟

الأنبوب يقرأ النوع من `@Body()` عبر `class-transformer` ثم `class-validator`. الحقول بلا مزخرفات زائدة. عند المنع يُرمى `BadRequestException` قبل المتحكم. الحراس يعملون حسب ترتيب Nest. إن سبق الحارس الأنبوب، قد تُرفض الهوية قبل الشكل. الشكل الزائد ما زال لا يصل إلى Prisma.

الأنبوب لا يرى سلسلة تُبنى لاحقًا في `$queryRawUnsafe`. ولا يغيّر `title` إلى نص مهرَّب.

مخطط Zod على `@Body({ schema })` مسار بديل موثّق للتحقق. الفكرة واحدة: عقد إدخال ضيق. هذه الدورة تبقي `class-validator` على موارد المقالات.

## مثال عملي

اختبار `POST /users` بجسم فيه `role` يتوقع `400` وغياب صف محرر. اختبار ثانٍ بجسم نظيف يتوقع `role` في القاعدة `reader` حتى لو أُعيد إرسال الطلب. اختبار ثالث يمرّر `password` بطول 3 ويتوقع `400` قبل `hash`. لا تختبر الأمن بتعطيل الأنبوب في e2e.

حدّث `CreatePostDto` بلا `authorId`، كما في فصل المستخدم الحالي. المؤلف من `@CurrentUser()`. وجود `authorId` في DTO يعيد التعيين الجماعي حتى مع whitelist، لأن الحقل «مسموح» بالخطأ.

## أخطاء شائعة

- `whitelist` بلا `forbid` ثم تجاهل أن الحقول اختفت بصمت في عميل قديم.
- نسخ `...dto` إلى `data` وفيها حقل نُسي.
- اعتبار `400` إثباتًا أن المستخدم موثّق.
- الاعتماد على الأنبوب ضد SQL أو XSS.
- إيقاف الأنبوب على مسار webhook بلا عقد بديل.

## متى نستخدمه؟

على كل جسم واستعلام عام يدخله عميل، كحد أول للشكل.

## متى لا نستخدمه؟

كسياسة ملكية المقال. الأنبوب لا يعرف هل Ada تملك `post/3`. ذلك `@Can` أو `authorize`. ولا كتشفير لكلمة المرور.

## مقارنة

| DTO التسجيل | جسم حر |
| --- | --- |
| بريد وكلمة فقط | `role` يصل إلى الخدمة |
| `400` عند حقل زائد | التعيين الجماعي |
| الدور في الخدمة | الدور من العميل |

## تمارين

### تمرين سهل

أي خيار يرفض الحقل غير المزخرف بدل حذفه؟

<details>
<summary>الحل</summary>

`forbidNonWhitelisted: true` مع `whitelist: true`.

</details>

### تمرين متوسط

لماذا يبقى `authorId` في `CreatePostDto` مشكلة حتى مع الأنبوب؟

<details>
<summary>الحل</summary>

الحقل المزخرف مسموح. العميل يختار المؤلف. المصدر الصحيح `@CurrentUser()`.

</details>

### تمرين متقدم

التحقق نجح و `title` سليم، ثم الخدمة تبني SQL بلصق `title`. أي طبقة فشلت؟

<details>
<summary>الحل</summary>

طبقة الاستعلام. الأنبوب فحص الشكل فقط. القيمة تذهب بمعامل أو `findMany`.

</details>

## أسئلة مراجعة

- ماذا يفعل `whitelist`؟
- من يكتب `role` عند التسجيل؟
- هل `400` يعني فشل الهوية؟

## الخلاصة

الأنبوب يضيق العقد ويرفض التعيين الجماعي للحقول التي لم تعلنها. الدور والتجزئة وشرط الملكية تبقى في الخدمة والسياسة. التهريب ومعاملات SQL تبقى في طبقاتهما.

## ماذا بعد؟

[Encryption و Hashing](./08-encryption-and-hashing.md) متى تُستعاد البيانات ومتى تُقارن فقط.

## المصادر الرسمية

- NestJS — Validation: https://docs.nestjs.com/application/validation
- NestJS — Authentication: https://docs.nestjs.com/security/authentication
