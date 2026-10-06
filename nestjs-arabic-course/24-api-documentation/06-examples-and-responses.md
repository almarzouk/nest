# Examples و Response Types

## الهدف من هذا الدرس

ربط كل عملية بنموذج الاستجابة وبأمثلة يملؤها Swagger، بما فيها قائمة المقالات المغلفة.

## المشكلة التي يحلها

`@ApiOkResponse()` بلا `type` يقول «نجاح» ولا يقول إن الجسم `{ id, title, published }` بلا `passwordHash`. مولد العميل يعيد `any`. قائمة المقالات تعيد `{ items, page, pageSize, total }` بينما الوثيقة تذكر مصفوفة عارية، فيكسر الربط في الواجهة.

## الفكرة ببساطة

صنف الاستجابة يحمل `@ApiProperty` على ما يخرج فقط. المزخرف `type` على `@ApiOkResponse` أو `@ApiCreatedResponse` يربط العملية بالصنف. الأمثلة في `example` أو `examples` تملأ النموذج. التغليف العام للصفحة يُركب بـ `allOf` و `getSchemaPath` لأن TypeScript لا يحتفظ بمعامل النوع.

## التعريف التقني

```ts
export class PostResponseDto {
  @ApiProperty()
  id: number;

  @ApiProperty({ example: "Sunday shelves" })
  title: string;

  @ApiProperty()
  published: boolean;
}
```

```ts
@ApiCreatedResponse({ type: PostResponseDto })
@Post()
create(@Body() dto: CreatePostDto): Promise<PostResponseDto> {
  return this.posts.create(dto);
}
```

هذا يوثّق الشكل. إخفاء `passwordHash` وقت التشغيل يبقى في `select` أو في التسلسل، لا في المزخرف وحده. إن أعاد المعالج كيانًا فيه التجزئة فالعميل يراها مهما قالت الوثيقة.

صفحة العمليات تعرض غلافًا عامًا: صنف فيه `total` و `limit` و `offset` و `results` بلا مزخرف على `results`، ثم `@ApiOkResponse` بمخطط `allOf` يضم `getSchemaPath(PaginatedDto)` ومصفوفة `getSchemaPath(CatDto)`. `@ApiExtraModels(PaginatedDto)` يلزم لأن لا معاملًا يشير إلى الغلاف مباشرة.

مزخرف قابل لإعادة الاستخدام من الصفحة، بصورة المكتبة:

```ts
export const ApiPaginatedResponse = <TModel extends Type<any>>(model: TModel) =>
  applyDecorators(
    ApiExtraModels(PageDto, model),
    ApiOkResponse({
      schema: {
        title: `PaginatedResponseOf${model.name}`,
        allOf: [
          { $ref: getSchemaPath(PageDto) },
          {
            properties: {
              items: { type: "array", items: { $ref: getSchemaPath(model) } },
            },
          },
        ],
      },
    }),
  );
```

`title` يسمي النوع `PaginatedResponseOfPostResponseDto`. الصفحة تذكر هذا مع مولّدات العميل. `Type` و `applyDecorators` من `@nestjs/common`، و `getSchemaPath` من `@nestjs/swagger`. حقول الغلاف هنا `items` و `page` و `pageSize` و `total`. `examples` على الخاصية كائن مفاتيحه أسماء وقيمه `{ value }`. `addGlobalResponse` لـ `500` يبقى في الباني.

## Mental Model

```text
صنف يخرج للعميل
  type على مزخرف الحالة
غلاف صفحة
  allOf + getSchemaPath + title
المثال يملأ الواجهة ولا يغيّر JSON الحقيقي
```

## أبسط مثال

```ts
@Get(":id")
@ApiOkResponse({ type: PostResponseDto })
@ApiNotFoundResponse({ description: "Unknown id" })
findOne(@Param("id") id: string) {
  return this.posts.findOne(id);
}
```

## شرح المثال

النجاح `200` بصنف المقال. الغياب وصف بلا صنف ثقيل، أو بصنف خطأ مشترك إن وحّدت شكل `HttpException`. لا تضع `PostResponseDto` على `404`. القارئ عندها يجهّز `title` في الخطأ.

## ماذا يحدث داخليًا؟

`createDocument` يضيف `PostResponseDto` إلى `components.schemas` لأنه مذكور في `type`. الاستجابة في المسار تشير إليه بـ `$ref`. `allOf` يدمج الغلاف مع مصفوفة العناصر. الواجهة تعرض مثال `Sunday shelves` من `example` إن لم يملأ المجرب الحقل. الاستجابة الحية تأتي من Prisma لا من المثال.

إن اختلف اسم الحقل في الصنف عن JSON بعد التسلسل، وثّق الناتج النهائي لا صف القاعدة.

## مثال عملي

`GET /posts` يستخدم `@ApiPaginatedResponse(PostResponseDto)`. افتح `api-json` وابحث عن `PaginatedResponseOfPostResponseDto` وعن `$ref` إلى `PostResponseDto`. تأكد أن `passwordHash` غير موجود في المخطط. `POST /posts` يبقى `@ApiCreatedResponse` لا `@ApiOkResponse`. تعليق منشأ يعيد `CommentResponseDto` بـ `id` و `body` و `postId` من غير بريد المؤلف إن لم يكن جزءًا من العقد.

## أخطاء شائعة

- `type: Post` حيث الصنف كيان القاعدة وفيه التجزئة.
- غلاف صفحة بلا `title` ثم عميل مولَّد بنوع مسطح غير مسمّى.
- نسيان `@ApiExtraModels` فيختفي `$ref`.
- مثال يحتوي بريدًا حقيقيًا.

## متى نستخدمه؟

على كل نجاح يرجع جسمًا، وعلى القوائم المغلفة حتى يتفق المولد مع الواجهة.

## متى لا نستخدمه؟

لاختيار الحقول في وقت التشغيل. صنف الوثيقة الذي يشمل سرًا لا يحذفه من الاستجابة.

## مقارنة

| `type: PostResponseDto` | `schema` مع `allOf` |
| --- | --- |
| كائن واحد | دمج غلاف مع مصفوفة |
| يكفي للقراءة والإنشاء | لازم للتعميم على الصفحة |
| الصنف يُكتشف من المزخرف | الغلاف يحتاج `ApiExtraModels` |

## تمارين

### تمرين سهل

أي خيار يربط `201` بصنف المقال؟

<details>
<summary>الحل</summary>

`@ApiCreatedResponse({ type: PostResponseDto })`.

</details>

### تمرين متوسط

لماذا `getSchemaPath` في غلاف الصفحة؟

<details>
<summary>الحل</summary>

ليشير المخطط إلى نموذج مسجّل في `components` عبر `$ref` بدل تكرار الخصائص.

</details>

### تمرين متقدم

ماذا يغيّر `title` داخل مخطط `ApiPaginatedResponse`؟

<details>
<summary>الحل</summary>

يعطي المولد اسم نوع مثل `PaginatedResponseOfPostResponseDto` بدل كائنًا مجهول الحقول.

</details>

## أسئلة مراجعة

- من أين تأتي قيمة المثال في الطلب الحي؟
- لماذا لا يُوثَّق كيان فيه `passwordHash` كاستجابة؟
- ماذا يلزم صنف الغلاف غير المذكور في `@Body`؟

## الخلاصة

`type` يسمي جسم النجاح. القوائم المغلفة تُركَّب بـ `allOf` و `getSchemaPath` وعنوان للمولد. الأمثلة للواجهة، والحقول الحساسة تبقى خارج صنف الاستجابة.

## ماذا بعد؟

[CLI Plugin](./07-cli-plugin.md) يملأ `@ApiProperty` من الأنواع والتعليقات.

## المصادر الرسمية

- NestJS — Operations: https://docs.nestjs.com/openapi/operations
- NestJS — Types and parameters: https://docs.nestjs.com/openapi/types-and-parameters
