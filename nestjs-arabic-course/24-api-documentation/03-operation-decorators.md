# Decorators الخاصة بالعمليات

## الهدف من هذا الدرس

تعليم كل عملية مقالات وسمًا وملخصًا واستجابات يراها قارئ الوثيقة قبل تجربة الطلب.

## المشكلة التي يحلها

`SwaggerModule` يسرد المسارات، لكن `POST /posts` يظهر بلا شرح، و `409` عند تكرار البريد غير مذكور، فيظن العميل أن الفشل الوحيد `400`. الوسوم الافتراضية من اسم الصنف تفرّق `Posts` عن `Comments` بشكل آلي قد لا يطابق لغة المنتج.

## الفكرة ببساطة

`@ApiTags` يجمع المتحكم. `@ApiOperation` يضع ملخص العملية. `@ApiResponse` ومختصراته يعلنون الحالات التي تعرفها أنت. `@ApiHeader` يوثّق رأسًا تتوقعه. المزخرف يغيّر الوثيقة لا سلوك Nest. رمز الحالة الحقيقي يبقى من `@HttpCode` أو من الاستثناء.

## التعريف التقني

```ts
import { Controller, Get, Param } from "@nestjs/common";
import {
  ApiNotFoundResponse,
  ApiOkResponse,
  ApiOperation,
  ApiTags,
} from "@nestjs/swagger";

@ApiTags("posts")
@Controller("posts")
export class PostsController {
  @Get(":id")
  @ApiOperation({ summary: "Read one post" })
  @ApiOkResponse({ description: "Published post, or the owner draft" })
  @ApiNotFoundResponse({ description: "Unknown id" })
  findOne(@Param("id") id: string) {
    return this.posts.findOne(id);
  }
}
```

مختصرات الصفحة تشمل `@ApiCreatedResponse` و `@ApiBadRequestResponse` و `@ApiUnauthorizedResponse` و `@ApiForbiddenResponse` و `@ApiConflictResponse` و `@ApiTooManyRequestsResponse`. اختر ما يرميه المسار فعلًا.

`DocumentBuilder.addGlobalResponse` يضيف حالة لكل العمليات، مثل `500`. لا تضع `404` عالميًا إن كانت بعض القوائم تعيد صفحة فارغة بـ `200`.

`@ApiHeader({ name, description })` على المتحكم إن كان كل مسار يحتاج الرأس. رأس الجلسة ليس هنا. الأمن له درس خاص. هذا الرأس لشيء مثل مفتاح تتبع داخلي إن وُجد.

`@ApiExtension('x-foo', value)` يضيف امتدادًا. الاسم بلا بادئة `x-` يرمي، حسب الصفحة.

وسوم هرمية عبر `addTag` مع `parent` و `kind` تتطلب `setOpenAPIVersion('3.2.0')`. `@ApiTags` يتجاهل تلك الحقول ويسجّل تحذيرًا. الهرم في `DocumentBuilder` فقط.

## Mental Model

```text
@ApiTags تجمع الصفحة
@ApiOperation سطر يشرح الفعل
@ApiOkResponse ما يعيده النجاح
الاستثناء في الكود هو مصدر الرمز
```

## أبسط مثال

```ts
@Post()
@ApiOperation({ summary: "Create a post for the current user" })
@ApiCreatedResponse({ description: "Post stored" })
@ApiBadRequestResponse({ description: "Validation failed" })
@ApiUnauthorizedResponse({ description: "Sign in required" })
create(@Body() dto: CreatePostDto) {
  return this.posts.create(dto);
}
```

## شرح المثال

الملخص يقول إن المؤلف هو المستخدم الحالي حتى لا يبحث القارئ عن `authorId` في الجسم. `201` يطابق `@Post` الناجح. `400` من الأنبوب. `401` من حارس المصادقة. `403` من السياسة يُذكر على التحديث لا على كل إنشاء إن كان الإنشاء للمحرر فقط. لا تعلن `204` إن كنت تعيد جسم المقال.

## ماذا يحدث داخليًا؟

المزخرفات تخزن بيانات وصفية يقرأها `createDocument` عند بناء `paths./posts.post.responses`. Swagger UI يعرضها تحت العملية. الطلب التجريبي لا يختار الاستجابة من المزخرف. الخادم يختارها. إن رميت `NotFoundException` بلا `@ApiNotFoundResponse` تبقى الوثيقة ناقصة والسلوك صحيحًا.

`@ApiOperation` لا يستبدل تعليقًا يقرأه الناس في الكود إلا إن فعّلت إضافة CLI لاحقًا. المزخرف صريح وواضح في هذا الدرس.

## مثال عملي

على `CommentsController` ضع الوسم `comments`. `POST /posts/:postId/comments` يأخذ `@ApiCreatedResponse` و `@ApiNotFoundResponse` حين يغيب المقال، و `@ApiUnauthorizedResponse` لأن التعليق يحتاج جلسة. `GET` العام يأخذ `@ApiOkResponse` فقط مع `400` إن فشل `postId`. راجع الوثيقة بعد التوليد وتأكد أن `409` يظهر على تسجيل البريد المكرر لا على إنشاء مقال بلا قيد فريد للعنوان.

## أخطاء شائعة

- نسخ `@ApiOkResponse` على `@Post` الذي يعيد `201`.
- إعلان `200` لمسار يرمي `404`.
- وسم لكل طريقة داخل متحكم واحد فيتفتت التصفح.
- الاعتقاد أن المزخرف يغيّر رمز Nest.

## متى نستخدمه؟

على كل عملية يستهلكها عميل خارج الوحدة، خصوصًا الحالات غير `200`.

## متى لا نستخدمه؟

لاختراع حالات لا يرميها الكود. الوثيقة الكاذبة أسوأ من الوثيقة الناقصة لأن العميل يجهّز فروعًا لا تحدث.

## مقارنة

| مختصر الحالة | `@ApiResponse` العام |
| --- | --- |
| اسم يوضح الرمز | تمرّر `status` بنفسك |
| مناسب لـ 201 و 404 و 401 | مناسب لرمز بلا مختصر |
| لا يغيّر التنفيذ | لا يغيّر التنفيذ |

## تمارين

### تمرين سهل

أي مزخرف يضع سطر «Create a post»؟

<details>
<summary>الحل</summary>

`@ApiOperation({ summary: "Create a post" })`.

</details>

### تمرين متوسط

أي مختصر يناسب تكرار البريد `409`؟

<details>
<summary>الحل</summary>

`@ApiConflictResponse`.

</details>

### تمرين متقدم

لماذا لا تضع `404` في `addGlobalResponse` لقوائم المقالات؟

<details>
<summary>الحل</summary>

الصفحة الفارغة في هذه الدورة `200` مع مصفوفة فارغة. `404` العالمي يعلّم العميل أن القائمة تختفي عند الفراغ.

</details>

## أسئلة مراجعة

- من أين يأتي رمز الحالة فعليًا؟
- ماذا يجمع `@ApiTags`؟
- متى يلزم `setOpenAPIVersion('3.2.0')` للوسوم؟

## الخلاصة

مزخرفات العملية تشرح الوسم والملخص والحالات. اجعلها مرآة للاستثناءات التي يرميها المسار، لا قائمة أمنيات.

## ماذا بعد؟

[توثيق DTO](./04-dto-documentation.md) خصائص الجسم في الوثيقة.

## المصادر الرسمية

- NestJS — Operations: https://docs.nestjs.com/openapi/operations
