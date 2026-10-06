# DTO و Validation

## الهدف من هذا الدرس

تثبيت جسم كل مسار كتابة في صنف يفحصه `ValidationPipe` قبل أن تصل الحقول إلى Prisma.

## المشكلة التي يحلها

`@Body() body: any` يمرّر `published` و `passwordHash` و `role` إن أرسلها العميل. نوع Prisma يُمحى وقت التشغيل فلا يرفض الحقل الزائد. مقال بلا عنوان يصل إلى `NOT NULL` كخطأ قاعدة بدل `400` يشرح الحقل.

## الفكرة ببساطة

صنف لكل أمر كتابة: `CreateUserDto` و `CreatePostDto` و `CreateCommentDto`. الأنبوب العام يبيّض الحقول غير الموسومة ويرفض الزائد ويحوّل الأنواع. الخدمة تستقبل الناتج لا الطلب الخام.

## التعريف التقني

في `main.ts`، كما في فصل التحقق:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

`whitelist` يحذف ما بلا وسوم. `forbidNonWhitelisted` يرمي `400` بدل الحذف الصامت. `transform` يبني نسخة من الصنف ويتيح `@Type` عند الحاجة.

```ts
import { IsEmail, IsInt, IsString, MaxLength, MinLength } from "class-validator";

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(1)
  @MaxLength(80)
  name: string;

  @IsString()
  @MinLength(12)
  @MaxLength(128)
  password: string;
}
```

`CreatePostDto` يضيف `@IsInt()` على `authorId`. مع `transform`، العدد القادم كنص JSON يبقى عددًا إن أرسله العميل عددًا. إن أرسل `"3"` كنص، التحويل الضمني يحتاج `transformOptions.enableImplicitConversion` بحذر، أو `@Type(() => Number)`. فضّل أن يرسل العميل عددًا حقيقيًا في JSON.

`CreateCommentDto` حقل `body` بـ `@MaxLength(500)`.

لا تستخدم واجهة TypeScript وحدها. الأنبوب يحتاج صنفًا. مخطط Zod مع `StandardSchemaValidationPipe` بديل موثّق في Nest 12 إن كان المشروع كله على المخططات. هذا الفصل يبقي class-validator لأن الموارد الثلاثة أصناف.

## Mental Model

```text
JSON
  -> ValidationPipe
       زائد -> 400
       عنوان فارغ -> 400
       نسخة CreatePostDto
  -> PostsService
  -> Prisma
```

## أبسط مثال

```ts
import { IsInt, IsString, MinLength } from "class-validator";

export class CreatePostDto {
  @IsString()
  @MinLength(1)
  title: string;

  @IsString()
  @MinLength(1)
  body: string;

  @IsInt()
  authorId: number;
}
```

```ts
@Post()
create(@Body() dto: CreatePostDto) {
  return this.posts.create(dto);
}
```

## شرح المثال

المتحكم لا يعيد فحص `title`. الأنبوب فعل ذلك. الخدمة ما زالت تفرض `published: false` لأن ذلك قاعدة لا شكل حقل. التحقق يمنع الشكل الكاذب. الخدمة تمنع المعنى الذي لا يُعبَّر بوسم عام، مثل «المؤلف موجود».

`password` في DTO المستخدم لا يُسمّى `passwordHash`. الاسم في العقد هو ما يكتبه الإنسان. الترجمة إلى عمود التجزئة تتم في الخدمة.

## ماذا يحدث داخليًا؟

الأنبوب يعمل بعد الوسطاء وقبل الحارس إن وُجد، ضمن دورة الطلب التي درستها. فشل التحقق استثناء يصل إلى مرشح الاستثناءات كـ `400` ورسالة فيها قائمة. الخدمة لا ترى الجسم الأصلي. الحقول المحذوفة مع `whitelist` وحدها تختفي بصمت، لذلك `forbidNonWhitelisted` أوضح لمن يخطئ في اسم الحقل.

## مثال عملي

أرسل `POST /posts` مع `{ "title": "", "body": "x", "authorId": 1, "published": true }`. فراغ العنوان `400`، وإن أصلحته فالحقل `published` زائد ويعيد `400` مع `forbidNonWhitelisted`. أرسل `{ "title": "وحدات", "body": "شرح", "authorId": 1 }` وتوقع وصول الثلاثة فقط إلى الخدمة. كرّر الفكرة على تعليق أطول من `500` حرف.

## أخطاء شائعة

- أنبوب بلا `whitelist`.
- DTO يعكس كل أعمدة الجدول بما فيها `id` و `passwordHash`.
- فحص «المؤلف موجود» داخل الواسم.
- نسيان الأنبوب في `main.ts` ثم الاعتقاد أن الوسوم تعمل وحدها.

## متى نستخدمه؟

على كل `@Body()` ولاحقًا على استعلامات القائمة في دروس التصفية.

## متى لا نستخدمه؟

داخل `PrismaService`. العميل المولَّد لا يفحص HTTP. الفحص على الحدود.

## مقارنة

| DTO | نموذج Prisma |
| --- | --- |
| شكل الطلب | شكل الصف |
| وسوم وقت التشغيل | أنواع وقت الترجمة |
| يرفض الزائد | يكتب ما تمرّره أنت |

## تمارين

### تمرين سهل

ماذا يفعل `forbidNonWhitelisted` مع حقل `role` على `CreateUserDto`؟

<details>
<summary>الحل</summary>

يرفض الطلب بـ `400` لأن `role` بلا وسم على الصنف.

</details>

### تمرين متوسط

لماذا لا يكون `published` في `CreatePostDto`؟

<details>
<summary>الحل</summary>

المسودة قرار خدمة عند الإنشاء. وجود الحقل في DTO يجعل العميل يختاره حتى لو فحصت القيمة لاحقًا.

</details>

### تمرين متقدم

الفريق يريد Zod لكل الأجسام. أين يتغيّر الأنبوب من غير نقل SQL إلى المتحكم؟

<details>
<summary>الحل</summary>

استبدل الأنبوب العام بـ `StandardSchemaValidationPipe` ومرّر المخطط على `@Body` كما في توثيق التحقق. الخدمة تبقى تستقبل كائنًا ضيقًا. لا تخلط الأنبوبين على التطبيق نفسه.

</details>

## أسئلة مراجعة

- أين تُفعَّل الخيارات الثلاثة؟
- ما الفرق بين اسم `password` واسم العمود؟
- أي فحص لا يخص DTO؟

## الخلاصة

كل كتابة لها DTO، والأنبوب العام يرفض الناقص والزائد قبل Prisma. قاعدة الوجود والمعنى تبقى في الخدمة.

## ماذا بعد؟

[Error Handling](./06-error-handling.md) يوحّد شكل `404` و `409` و `400`.

## المصادر الرسمية

- NestJS — Validation: https://docs.nestjs.com/application/validation
- NestJS — Pipes: https://docs.nestjs.com/pipes
