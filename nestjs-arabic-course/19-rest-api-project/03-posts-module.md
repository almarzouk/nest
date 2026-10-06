# Posts

## الهدف من هذا الدرس

بناء مورد المنشور: إنشاء مقال لمؤلف موجود، وجلب مقال واحد، من غير قائمة بعد.

## المشكلة التي يحلها

`POST /posts` بجسم فيه `authorId` قد يشير إلى مستخدم محذوف. كتابة المنشور داخل `UsersController` تجعل مسار المستخدم يعرف أعمدة المقال. القائمة ستحتاج تصفية وترتيبًا، فتُفصل عن الإنشاء حتى لا يكبر المتحكم دفعة واحدة.

## الفكرة ببساطة

`PostsService.create` تتأكد أن المؤلف موجود ثم تدرج `Post`. `findOne` تجلب المقال أو ترمي `404`. الحقول العامة: `id` و `title` و `body` و `published` و `authorId`. جسم الإنشاء لا يحدّد `published` في هذا الدرس: المقال يبدأ مسودة. النشر دالة `publish` لاحقًا أو حقل في تحديث منفصل.

## التعريف التقني

```ts
@Controller("posts")
export class PostsController {
  constructor(private readonly posts: PostsService) {}

  @Post()
  create(@Body() dto: CreatePostDto) {
    return this.posts.create(dto);
  }

  @Get(":id")
  findOne(@Param("id", ParseIntPipe) id: number) {
    return this.posts.findOne(id);
  }
}
```

`CreatePostDto`: `title` نص غير فارغ، `body` نص، `authorId` عدد صحيح. الخدمة:

```ts
await this.users.findOne(input.authorId);
return this.prisma.post.create({
  data: {
    title: input.title,
    body: input.body,
    authorId: input.authorId,
    published: false,
  },
  select: { id: true, title: true, body: true, published: true, authorId: true },
});
```

`UsersModule` مصدّرة لـ `UsersService`، و `PostsModule` تستوردها مع `PrismaModule`.

مسار بديل `GET /users/:userId/posts` قراءة فقط، ويمكن أن يعيش في `PostsController` بمسار أو في متحكم فرعي. الإنشاء يبقى `POST /posts` حتى يكون للمنشور معرّف مورد واضح. كلا التصميمين REST مقبول إن ثبتّ واحدًا.

## Mental Model

```text
CreatePostDto
   -> UsersService.findOne(authorId)   404 إن غاب
   -> prisma.post.create               published false
GET /posts/3
   -> findUnique أو 404
```

## أبسط مثال

```ts
import { Injectable, NotFoundException } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service.js";
import { UsersService } from "../users/users.service.js";

@Injectable()
export class PostsService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly users: UsersService,
  ) {}

  async create(input: { title: string; body: string; authorId: number }) {
    await this.users.findOne(input.authorId);
    return this.prisma.post.create({
      data: { ...input, published: false },
      select: {
        id: true,
        title: true,
        body: true,
        published: true,
        authorId: true,
      },
    });
  }

  async findOne(id: number) {
    const post = await this.prisma.post.findUnique({
      where: { id },
      select: {
        id: true,
        title: true,
        body: true,
        published: true,
        authorId: true,
      },
    });
    if (!post) {
      throw new NotFoundException("Post not found");
    }
    return post;
  }
}
```

## شرح المثال

`findOne` على المستخدم ترمي `404` برسالة المستخدم إن غاب المؤلف، قبل `INSERT`. يمكن تضييق الرسالة إلى «المؤلف غير موجود» بدالة `requireAuthor` حتى لا يظن العميل أن المسار خطأ. `published: false` قرار الخدمة لا حقل يثق بالعميل.

انتشار الكائن `input` آمن هنا لأن النوع يضم ثلاثة حقول فقط. لو كان `dto` صنفًا فيه حقول زائدة، عدّد الحقول في `data`.

## ماذا يحدث داخليًا؟

استعلامان في الإنشاء: قراءة مؤلف ثم إدراج. هذا أوضح من الاعتماد على خطأ المفتاح الأجنبي وحده، ويثمن رحلة إضافية. تحت تزامن حذف المؤلف بين الخطوتين، القيد ما زال يرفض الإدراج. امسك ذلك الخطأ إن ظهر وحوّله إلى `404` أو `409` حسب ما تعد به.

`GET` لا يكتب. إعادة الطلب لا تغيّر صفًا.

## مثال عملي

أنشئ مستخدم `1` ثم `POST /posts` بعنوان `Nest modules` و `authorId: 1`. الاستجابة `201` من `@Post` افتراضيًا في Nest، وفيها `published: false`. `authorId: 404040` يعيد `404` من خدمة المستخدم. `GET /posts/1` يعيد المقال. القائمة العامة تأتي في درس الترقيم على نفس الخدمة بدالة `list`.

## أخطاء شائعة

- قبول `published: true` من الجسم في مسار المسودة.
- SQL المنشور داخل `UsersController`.
- نسيان استيراد `UsersModule`.
- إرجاع تعليقات المقال في كل `GET` قبل أن يوجد عقد لذلك.

## متى نستخدمه؟

لكل مقال له مؤلف صف في `User`.

## متى لا نستخدمه؟

لنشر عام بلا مسودة. أضف عملية نشر صريحة حين يكون للفعل معنى مختلف عن التعديل.

## مقارنة

| فحص المؤلف في الخدمة | انتظار خطأ القاعدة فقط |
| --- | --- |
| رسالة يمكن تسميتها | `P2003` خام إن لم تترجمه |
| استعلام إضافي | رحلة واحدة قد تفشل |

## تمارين

### تمرين سهل

من يضبط `published` عند الإنشاء؟

<details>
<summary>الحل</summary>

`PostsService` تضعه `false`. العميل لا يرسله في `CreatePostDto`.

</details>

### تمرين متوسط

لماذا لا تُدرج المقالات من `UsersService.create`؟

<details>
<summary>الحل</summary>

إنشاء الحساب ليس إنشاء مقال. كل مورد يبقى في وحدته، والرابط `authorId`.

</details>

### تمرين متقدم

احذف المستخدم بين `findOne` و `create`. أي حاجز يمنع مقالًا يتيمًا؟

<details>
<summary>الحل</summary>

المفتاح الأجنبي `authorId`. ترجم خطأ Prisma إن وصل إلى مسار شائع، ولا تعتمد على النافذة الزمنية وحدها.

</details>

## أسئلة مراجعة

- ما حقول الاستجابة للمنشور؟
- لماذا تُستورد `UsersModule`؟
- ماذا يرجع مسار جلب مقال غير موجود؟

## الخلاصة

المنشور مورد في وحدته، المؤلف يُفحص عبر خدمة المستخدمين، والمسودة قرار خدمة لا حقل حر في الجسم.

## ماذا بعد؟

[Comments](./04-comments-module.md) يعلّق على منشور موجود.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- NestJS — Modules: https://docs.nestjs.com/modules
