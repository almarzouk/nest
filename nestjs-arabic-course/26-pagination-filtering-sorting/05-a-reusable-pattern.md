# نمط قابل لإعادة الاستخدام

## الهدف من هذا الدرس

جمع الصفحة والمرشح والفرز في أنواع صغيرة تستخدمها المقالات والتعليقات من غير نسخ DTO في كل متحكم.

## المشكلة التي يحلها

`ListPostsQueryDto` في الفصل 19 صحيحة لمورد واحد. نسخها إلى التعليقات ثم إلى القروض لاحقًا تغيّر السقف في نسخة وتنسى `forbidNonWhitelisted` في أخرى. غلاف JSON مرة `items` ومرة `results`. Swagger يوثّق كل غلاف يدويًا بشكل مختلف.

## الفكرة ببساطة

قاعدة استعلام مشتركة: `page` و `pageSize`. كل مورد يمدها بمرشحاته ومفاتيح فرزه. دالة `toSkipTake` ودالة `pageOf` تبنيان الغلاف. التعليقات الطويلة تستخدم نوع مؤشر منفصل `CursorQuery` حتى لا يختلط `page` مع `cursor` على المسار نفسه. التوثيق يستخدم `ApiPaginatedResponse` من فصل OpenAPI.

## التعريف التقني

```ts
import { Type } from "class-transformer";
import { IsInt, IsOptional, Max, Min } from "class-validator";

export class PageQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1)
  page = 1;

  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(50)
  pageSize = 20;
}

export function toSkipTake(query: PageQueryDto) {
  return {
    skip: (query.page - 1) * query.pageSize,
    take: query.pageSize,
  };
}

export function pageOf<T>(query: PageQueryDto, items: T[], total: number) {
  return {
    items,
    page: query.page,
    pageSize: query.pageSize,
    total,
  };
}
```

مقالات:

```ts
export class ListPostsQueryDto extends PageQueryDto {
  @IsOptional()
  @IsIn(POST_SORTS)
  sort: (typeof POST_SORTS)[number] = "id";

  @IsOptional()
  @IsIn(["asc", "desc"])
  order: "asc" | "desc" = "asc";
}
```

أضف `published` بمحلّل صريح، لا بتحويل ضمني. التعليقات على الإزاحة تستطيع وراثة `PageQueryDto` فقط، مع `postId` من المسار. خيط المؤشر صنف آخر:

```ts
export class CommentCursorQueryDto {
  @IsOptional() @IsString()
  cursor?: string;

  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(50)
  limit = 20;
}
```

الخدمة تبني `where` من `visibility` ثم `orderByPosts` ثم `toSkipTake`، وتجمع `findMany` مع `count` على الكائن نفسه، وتمرّر الصفوف على `toPostResponse` قبل `pageOf`. المتحكم يوثّق الناتج بـ `@ApiPaginatedResponse(PostResponse)`.

`ValidationPipe` مع `transform` و `whitelist` و `forbidNonWhitelisted` يطبّق على الصنف الوريث. حقل المقالات على مسار التعليقات يُرفض لأن DTO التعليق لا تعرفه. الصنف الوريث هو الذي يوثّق مفاتيح الفرز في OpenAPI.

## Mental Model

```text
PageQueryDto مشترك
  صنف مورد: مرشح + فرز
  toSkipTake + pageOf
  toResponse لكل عنصر
مسار المؤشر صنف ثانٍ
```

## أبسط مثال

```ts
return pageOf(query, items, total);
```

## شرح المثال

كل القوائم المرقّمة تمر من هنا فيصبح المفتاح `items` مضمونًا. اختبار `page=2` و `pageSize=20` يتوقع `skip=20`. وراثة الصنف تبقي مزخرفات الأب، والقيم الافتراضية تُورَّث مع الابن.

## ماذا يحدث داخليًا؟

الأنبوب يبني `ListPostsQueryDto` ويملأ الافتراض. الخدمة لا تعيد قراءة `req.query`. `@Type` يحوّل الأرقام قبل Prisma. الغلاف كائن عادي. عناصره تأتي من `toPostResponse`. `ApiPaginatedResponse` يسجّل الغلاف بعنوان يتضمن اسم العنصر، وحقوله `items` كما في درس الاستجابات.

## مثال عملي

انقل منطق `GET /posts` من DTO الفصل 19 إلى هذه الأنواع بلا تغيير أسماء الاستعلام، حتى لا تنكسر الواجهة. أضف `GET /posts/:postId/comments` بـ `PageQueryDto` أو بمؤشر، لا بالاثنين. اختبار e2e واحد يتوقع المفاتيح `items` و `page` و `pageSize` و `total` على المقالات. مسار المؤشر يتوقع `nextCursor` ولا يتوقع `page`.

## أخطاء شائعة

- نسخ DTO مع `@Max(100)` في مورد واحد.
- جمع `cursor` و `page` في صنف واحد.
- `pageOf` على صف Prisma قبل `toPostResponse`.
- قراءة `req.query.page` بعد الأنبوب.

## متى نستخدمه؟

ابتداءً من المورد الثاني الذي يعيد قائمة، وعند لمس قائمة المقالات مرة أخرى.

## متى لا نستخدمه؟

لإجبار المؤشر على شكل الصفحة المرقّمة. المساران نوعان. ولا كطبقة تخفي `where` الخام الذي يمر من العميل.

## مقارنة

| `PageQueryDto` أب | DTO منسوخ |
| --- | --- |
| السقف في مكان واحد | السقف ينحرف |
| `pageOf` شكل واحد | مفاتيح مختلفة |
| الابن يضيف مرشح المورد | كل ملف يعيد المحلّل المنطقي |

## تمارين

### تمرين سهل

ماذا تعيد `toSkipTake` للصفحة `1` والحجم `20`؟

<details>
<summary>الحل</summary>

`skip: 0` و `take: 20`.

</details>

### تمرين متوسط

لماذا لا يرث `CommentCursorQueryDto` من `PageQueryDto`؟

<details>
<summary>الحل</summary>

العقدان مختلفان: رقم صفحة مقابل مؤشر. جمعها في صنف واحد يجعل الاستعلام غامضًا.

</details>

### تمرين متقدم

كيف تبقى عناصر الغلاف بلا `passwordHash`؟

<details>
<summary>الحل</summary>

`toPostResponse` أو `UserResponse` قبل `pageOf`، مع `select` ضيق. الغلاف لا ينظّف صفوفًا خامًا وحده.

</details>

## أسئلة مراجعة

- أين يعيش `@Max(50)`؟
- أي دالة تبني `items` و `total`؟
- ماذا يوثّق قائمة المقالات في Swagger؟

## الخلاصة

أب للاستعلام المرقّم، وابن لمرشحات المورد، و`pageOf` للغلاف، وصنف مؤشر منفصل للخيط الطويل. العناصر أصناف استجابة.

## ماذا بعد؟

[أثر النمط على الاستعلام](./06-database-consequences.md) الفهارس وعمق الإزاحة.

## المصادر الرسمية

- NestJS — Validation: https://docs.nestjs.com/application/validation
- NestJS — OpenAPI operations: https://docs.nestjs.com/openapi/operations
- Prisma — Pagination: https://www.prisma.io/docs/orm/prisma-client/queries/pagination
