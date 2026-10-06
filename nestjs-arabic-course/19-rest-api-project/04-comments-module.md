# Comments

## الهدف من هذا الدرس

إضافة التعليق كمورد تابع لمقال: يُنشأ على مسار المقال ويُسرد معه، ويبقى في `CommentsModule`.

## المشكلة التي يحلها

تعليق بلا `postId` صالح يعلق نصًا في الهواء. وضع مصفوفة تعليقات داخل عمود JSON في `Post` يمنع قيد الطول ويمنع جلب تعليق واحد. كتابة التعليق من `PostsService` يجعل وحدة المقال تعرف كل أعمدة النقاش.

## الفكرة ببساطة

`POST /posts/:postId/comments` ينشئ تعليقًا بعد التأكد أن المقال موجود. `GET /posts/:postId/comments` يعيد تعليقات ذلك المقال فقط. المعرّف في المسار مصدر `postId`، لا حقل في الجسم يمكن أن يخالفه.

## التعريف التقني

```ts
@Controller("posts/:postId/comments")
export class CommentsController {
  constructor(private readonly comments: CommentsService) {}

  @Post()
  create(
    @Param("postId", ParseIntPipe) postId: number,
    @Body() dto: CreateCommentDto,
  ) {
    return this.comments.create(postId, dto.body);
  }

  @Get()
  list(@Param("postId", ParseIntPipe) postId: number) {
    return this.comments.listForPost(postId);
  }
}
```

`CreateCommentDto` حقل `body` فقط، بحد أدنى حرف وحد أقصى تحدده، مثل `500`.

الخدمة تستدعي `PostsService.findOne(postId)` ثم:

```ts
return this.prisma.comment.create({
  data: { body, postId },
  select: { id: true, body: true, postId: true },
});
```

`CommentsModule` تستورد `PostsModule` و `PrismaModule`. لذلك `PostsModule` تصدّر `PostsService`. تجنّب أن تستورد `PostsModule` خدمة التعليقات في الاتجاهين. القائمة على مسار التعليقات، لا داخل `GET /posts/:id`، حتى لا يكبر مقال فيه ألف تعليق. إن أردت عدّادًا على المقال، أضفه لاحقًا باستعلام `count` لا بضمّ كل النصوص.

## Mental Model

```text
المسار /posts/3/comments
        |
 ParseIntPipe على 3
        |
 PostsService.findOne(3)
        |
 prisma.comment حيث postId = 3
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service.js";
import { PostsService } from "../posts/posts.service.js";

@Injectable()
export class CommentsService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly posts: PostsService,
  ) {}

  async create(postId: number, body: string) {
    await this.posts.findOne(postId);
    return this.prisma.comment.create({
      data: { postId, body },
      select: { id: true, body: true, postId: true },
    });
  }

  async listForPost(postId: number) {
    await this.posts.findOne(postId);
    return this.prisma.comment.findMany({
      where: { postId },
      orderBy: { id: "asc" },
      select: { id: true, body: true, postId: true },
      take: 50,
    });
  }
}
```

## شرح المثال

`take: 50` سقف مؤقت حتى لا تُسحب كل النقاشات في أول نسخة. الترقيم القابل لإعادة الاستخدام يأتي في الفصل `26`. هنا السقف يحمي المسار. `orderBy: { id: "asc" }` يعرض الأقدم أولًا، وهو قرار منتج لهذا المورد لا عقد ترتيب عام.

فحص المقال يوحّد `404` بين الإنشاء والقائمة. مقال غير موجود لا يعيد قائمة فارغة تتظاهر بالنجاح.

## ماذا يحدث داخليًا؟

المسار المركّب ما زال متحكمًا عاديًا. Nest يملأ `postId` من العنوان قبل الجسم. `ValidationPipe` يفحص `body` فقط. الخدمة لا تقرأ `req`. قيد المفتاح الأجنبي يمنع تعليقًا إن حُذف المقال بعد الفحص. احذف التعليقات مع المقال داخل `$transaction` حين تضيف مسار حذف المقال، وإلا يبقى قرار `onDelete` في المخطط هو من يحدد بقاء الصفوف.

## مثال عملي

على المقال `3` أرسل `{ "body": "شرح واضح" }`. الاستجابة `201` وفيها `postId: 3`. أرسل الجسم نفسه إلى `/posts/999/comments` وتوقع `404`. اطلب القائمة وتوقع التعليق فقط، لا صفوف مقال `4`. حاول إرسال `postId` داخل الجسم: الـ DTO لا يعرّفه، ومع `whitelist` و `forbidNonWhitelisted` يُرفض الحقل الزائد بـ `400`.

## أخطاء شائعة

- أخذ `postId` من الجسم ومن المسار معًا واحتمال اختلافهما.
- قائمة تعليقات بلا `where`.
- استيراد دائري بين `PostsModule` و `CommentsModule`.
- ضم كل التعليقات في `GET /posts/:id`.

## متى نستخدمه؟

حين للتعليق أب هو المقال، والمسار يسمي الأب.

## متى لا نستخدمه؟

لتعليق على تعليق في النسخة الأولى. أضف `parentId` حين يظهر السيناريو، لا في جدول اليوم.

## مقارنة

| عمود JSON داخل المقال | جدول `Comment` |
| --- | --- |
| بلا صف مستقل | `id` لكل تعليق |
| صعب مع قيد الطول | `body` عمود ونص DTO |
| يكبر صف المقال | استعلام منفصل بسقف |

## تمارين

### تمرين سهل

من أين تأتي قيمة `postId` عند الإنشاء؟

<details>
<summary>الحل</summary>

من معامل المسار `posts/:postId/comments` بعد `ParseIntPipe`، لا من الجسم.

</details>

### تمرين متوسط

لماذا قائمة مقال غير موجود `404` وليست `[]`؟

<details>
<summary>الحل</summary>

المسار يعد بوجود المقال. القائمة الفارغة تعني مقالًا بلا تعليقات. الغياب خطأ مختلف.

</details>

### تمرين متقدم

`PostsModule` تحتاج عدد التعليقات. كيف تتجنّب استيراد `CommentsModule` إن كانت تلك تستورد `PostsModule` أصلًا؟

<details>
<summary>الحل</summary>

استعلام `count` داخل `PostsService` عبر `PrismaService` على جدول التعليقات، أو وحدة ثالثة تقرأ الطرفين. لا تستورد وحدة التعليقات كاملة إن كان ذلك يصنع دورة.

</details>

## أسئلة مراجعة

- ما مسارا التعليق؟
- لماذا سقف `take`؟
- أي وحدة تصدّر خدمة المقال؟

## الخلاصة

التعليق مورد تابع يُعنوَن بمسار المقال، ويُكتب من وحدته بعد التأكد من الأب، بسقف قائمة مؤقت.

## ماذا بعد؟

[DTO و Validation](./05-dto-and-validation.md) يجمع أجسام الموارد الثلاثة تحت الأنبوب.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- NestJS — Validation: https://docs.nestjs.com/application/validation
