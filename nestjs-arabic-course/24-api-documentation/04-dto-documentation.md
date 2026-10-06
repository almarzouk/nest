# توثيق DTO

## الهدف من هذا الدرس

إظهار خصائص `CreatePostDto` وزملائه في OpenAPI عبر `@ApiProperty`، إلى أن تتولى إضافة CLI ذلك.

## المشكلة التي يحلها

`createDocument` يرى `@Body() dto: CreatePostDto` ويصنع نموذجًا باسم الصنف. الخصائص تبقى فارغة لأن TypeScript لا يحتفظ بها وقت التشغيل. Swagger UI يعرض جسمًا بلا حقول، فيرسل المجرب `{}` ويأخذ `400` من غير أن يفهم العقد.

## الفكرة ببساطة

`@ApiProperty` على كل حقل يُنشر في الوثيقة. `@ApiPropertyOptional` للحقل غير الإلزامي. المصفوفة تحتاج `type: [String]` أو `isArray`. التعداد يُمرَّر في `enum`، ومع `enumName` يصبح مخططًا يعاد استخدامه. الحقول التي لا يرسلها العميل، مثل `authorId`، لا تُوضع على DTO الإنشاء أصلًا.

## التعريف التقني

```ts
import { ApiProperty, ApiPropertyOptional } from "@nestjs/swagger";
import { IsBoolean, IsOptional, IsString, MaxLength, MinLength } from "class-validator";

export class CreatePostDto {
  @ApiProperty({ example: "Sunday shelves", minLength: 1, maxLength: 120 })
  @IsString()
  @MinLength(1)
  @MaxLength(120)
  title: string;

  @ApiPropertyOptional({ default: false })
  @IsOptional()
  @IsBoolean()
  published?: boolean;
}
```

`example` يظهر في الواجهة. لا تضع كلمة مرور حقيقية. الحدود في `@ApiProperty` توثيق. `@MinLength` هو الذي يرفض الطلب. الإضافة لاحقًا تستطيع نقل بعض قواعد `class-validator` إلى المخطط، وما زلت تبقي المزخرفات للأنبوب.

مصفوفة وسوم: `@ApiProperty({ type: [String] }) tags: string[]`. صفحة الأنواع تقول إن الإضافة تكتشف المصفوفة وحدها.

مرجع دائري: `@ApiProperty({ type: () => CommentDto })`. الإضافة تكتشف الدوران أيضًا.

`enum` مع `enumName: "PostSort"` يمنع تكرار التعداد في العملاء المولَّدين، كما تشرح الصفحة مع NSwag. `@ApiQuery({ name: "sort", enum: PostSort, enumName: "PostSort" })` للاستعلام.

`@ApiSchema({ name: "CreatePostRequest" })` يغيّر اسم المخطط في `components`. مفيد حين يختلف اسم الصنف عن لغة العقد.

`@ApiExtraModels` يسجّل صنفًا لا يظهر كـ `@Body` مباشر، مثل غلاف صفحة ستركّبه في درس الأمثلة. `getSchemaPath` يعيد `#/components/schemas/...`.

`@ApiBody({ type: [CreatePostDto] })` لإنشاء جماعي لأن المصفوفة العامة لا تُحفظ في الانعكاس.

`PartialType` المستخدم في DTO التحديث يُستورد من `@nestjs/swagger` حين تعمل الإضافة، حتى يلتقط المخطط الحقول الاختيارية. من غير الإضافة يمكنك توثيق `UpdatePostDto` بمزخرفات اختيارية مباشرة.

## Mental Model

```text
صنف TypeScript غير مرئي لـ Swagger
@ApiProperty يجعل الحقل مرئيًا
class-validator يبقى حارس الطلب
```

## أبسط مثال

```ts
export class CreateCommentDto {
  @ApiProperty({ maxLength: 500 })
  @IsString()
  @MaxLength(500)
  body: string;
}
```

## شرح المثال

التعليق جسم ونص فقط. `postId` في المسار لا في الجسم، لذلك لا خاصية له هنا. وثيقة `POST /posts/{postId}/comments` ستظهر المعامل من `@Param`، والجسم من هذا الصنف بعد المزخرف. أضف وصفًا قصيرًا إن كان الحد `500` غير واضح من الاسم.

## ماذا يحدث داخليًا؟

عند بناء الوثيقة تُقرأ البيانات الوصفية لـ `@ApiProperty` وتُكتب تحت `components.schemas.CreateCommentDto.properties.body`. الواجهة تبني نموذجًا يمكن ملؤه. الطلب إلى Nest لا يمر على `@ApiProperty`. `ValidationPipe` يقرأ `@MaxLength`. حذف مزخرف التحقق وإبقاء `@ApiProperty` يوثّق حدًا لا يُفرض.

حقل بلا مزخرف توثيق يبقى خارج النموذج حتى مع وجوده في الصنف، فيختلف Swagger عن السلوك إن كان `whitelist` يمرّر الحقل المزخرف للتحقق فقط. زامن الاثنين.

## مثال عملي

`ListPostsQueryDto` وثّق `page` و `pageSize` مع `minimum` و `maximum: 50` حتى ترى الواجهة السقف الذي يفرضه `@Max`. `sort` تعداد `id` و `title` مع `enumName`. لا توثّق `password` في أي استجابة. DTO الدخول يمكن أن يوثّق الحقل للتجربة في بيئة مغلقة، والمثال سلسلة واضحة أنها وهمية مثل `correct-horse-battery`.

افتح `/api` بعد إعادة التشغيل وتأكد أن النموذج لم يعد `{}`.

## أخطاء شائعة

- `@ApiProperty` بلا `@IsString` ثم ظن أن الوثيقة تتحقق.
- توثيق `authorId` لأن العمود موجود في الجدول.
- `enum` بلا `enumName` في عميل يولَّد فيتكرر النوع.
- نسيان `type: [String]` للمصفوفة قبل تفعيل الإضافة.

## متى نستخدمه؟

على كل DTO يدخل الوثيقة قبل أن تعتمد على إضافة CLI، وعلى أي حقل تريد مثاله أوحدّه ظاهرًا فوق ما تستنتجه الإضافة.

## متى لا نستخدمه؟

لإخفاء سلوك الأنبوب. إن رفضت الحقل في التحقق فاحذفه من العقد لا تكتفِ بإغفال المثال.

## مقارنة

| `@ApiProperty` | `@ApiPropertyOptional` |
| --- | --- |
| الحقل في `required` | `required: false` |
| إنشاء العنوان | `published` له افتراض |
| ما زال يحتاج محققًا | ما زال يحتاج `@IsOptional` |

## تمارين

### تمرين سهل

لماذا يظهر `CreatePostDto` فارغًا بلا مزخرفات؟

<details>
<summary>الحل</summary>

الانعكاس لا يرى خصائص TypeScript. `@ApiProperty` أو إضافة CLI تكتبانها في الوثيقة.

</details>

### تمرين متوسط

كيف توثّق مصفوفة وسوم نصية؟

<details>
<summary>الحل</summary>

`@ApiProperty({ type: [String] })` أو `isArray: true` مع النوع.

</details>

### تمرين متقدم

عميل NSwag كرّر تعداد الترتيب في كل DTO. ماذا تضيف؟

<details>
<summary>الحل</summary>

`enumName` بجانب `enum` حتى يُنشر المخطط مرة في `components.schemas`.

</details>

## أسئلة مراجعة

- أي حزمة تصدّر `@ApiProperty`؟
- هل المثال في المزخرف يفرض القيمة؟
- أين يُوثَّق `postId` للتعليق؟

## الخلاصة

`@ApiProperty` يملأ نموذج OpenAPI بما يسمح الانعكاس بإظهاره. التحقق يبقى على `class-validator`. الحقول التي لا يكتبها العميل لا تدخل DTO حتى تبدو قانونية في Swagger.

## ماذا بعد؟

[توثيق Auth](./05-auth-in-swagger.md) قفل الكعكة و `bearer` على العمليات.

## المصادر الرسمية

- NestJS — Types and parameters: https://docs.nestjs.com/openapi/types-and-parameters
