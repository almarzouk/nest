# @nestjs/authorization

## الهدف من هذا الدرس

تسجيل سياسات المكتبة على الحزمة الرسمية: `@Policy` و `@Can` و `AuthorizationService`.

## المشكلة التي يحلها

دوال السياسة التي كتبناها باليد تُنسى على مسار جديد، وتُعاد صياغتها في حارس مختلف عن الخدمة. الرمز عند الضيف يختلف عن الرمز عند المستخدم المعروف إن رمى كل مسار استثناءه الخاص.

## الفكرة ببساطة

`AuthorizationModule.forRoot()` بعد `AuthenticationModule`. الصنف عليه `@Policy()` ودواله abilities. `@Can(PostPolicy, "create")` يشغّل القدرة قبل المعالج إن كانت تُستدعى بالمستخدم وحده. القدرة التي تحتاج المقال تُستدعى في الخدمة عبر `authorize` أو `can`. `true` الحرفي فقط يسمح.

## التعريف التقني

ترتيب الوحدات من الصفحة: `AuthenticationModule` أولًا ثم `AuthorizationModule`. العكس يجعل حارس التفويض يرى ضيفًا قبل أن يضع حارس المصادقة `request.user`، فيرفض الإقلاع ويسمّي الحارسين.

```ts
@Module({
  imports: [
    AuthenticationModule.forRoot({ session: { absoluteTtl: "14d", idleTtl: "3d" } }),
    AuthorizationModule.forRoot(),
    PostsModule,
  ],
})
export class AppModule {}
```

```ts
import { Policy } from "@nestjs/authorization";

@Policy()
export class PostPolicy {
  create(user: PublicUser | null) {
    return !!user?.roles.includes("editor");
  }

  update(user: PublicUser | null, post: { authorId: number }) {
    return !!user && user.id === post.authorId;
  }
}
```

`@Can(PostPolicy, "create")` على `POST /posts` لأن `create` تقبل المستخدم فقط وتقبل `null`. `@Can(PostPolicy, "update")` لا يصلح على المسار: الحارس لا يملك المقال، والصفحة ترفض القدرة التي تحتاج سجلًا أو لا تقبل ضيفًا في هذا الموضع.

في الخدمة، بعد تحميل المقال:

```ts
await this.authorization.authorize(PostPolicy, "update", user, post);
```

`can` ترجع منطقيًا. `authorize` ترمي `AuthorizationError` بسبب `unauthenticated` أو `forbidden`، والحزمة تحوّله إلى `401` أو `403` على HTTP. في عامل طابور يبقى الخطأ نفسه لأن الخدمة ليست HTTP حصرًا.

اكتشف السياسات كمزوّدات في الوحدة، أو اسردها في خيار `policies`. `@Policy()` يجعل الصنف قابلًا للحقن. خطأ إملائي في اسم القدرة على `@Can` يفشل عند الإقلاع ويذكر القدرات الصالحة، كما في تحذير الصفحة عن اسم مثل `craete`.

`before()` على السياسة يسبق كل قدرة. أرجئه حتى تحتاج تجاوز مشرف، والصفحة تحذّر أنه يتخطى حتى القواعد التي يجب أن تسري على المشرف إن رجعت `true` دائمًا.

## Mental Model

```text
AuthenticationModule
AuthorizationModule
PostPolicy.create     -> @Can على المسار
PostPolicy.update     -> authorize في الخدمة بعد التحميل
true فقط              -> سماح
```

## أبسط مثال

```ts
import { Body, Controller, Post } from "@nestjs/common";
import { CurrentUser } from "@nestjs/authentication";
import { Can } from "@nestjs/authorization";
import { CreatePostDto } from "./create-post.dto.js";
import { PostPolicy } from "./post.policy.js";
import { PostsService } from "./posts.service.js";
import type { PublicUser } from "../users/public-user.js";

@Controller("posts")
export class PostsController {
  constructor(private readonly posts: PostsService) {}

  @Post()
  @Can(PostPolicy, "create")
  create(@CurrentUser() user: PublicUser, @Body() dto: CreatePostDto) {
    return this.posts.create(user.id, dto);
  }
}
```

## شرح المثال

المحرر يمر. القارئ المعروف يأخذ `403` لأن الهوية موجودة والقدرة رجعت غير `true`. الضيف على مسار غير عام أصلًا يوقفه حارس المصادقة بـ `401` قبل `@Can` إن كان المسار مغلقًا. إن فتحت المسار، `create` ترى `null` وترجع `false` فيصبح الرفض من التفويض.

عدّل المقال في الخدمة لا بمزخرف:

```ts
await this.authz.authorize(PostPolicy, "update", user, post);
```

## ماذا يحدث داخليًا؟

حارس التفويض العام يقرأ البيانات التي وضعها `@Can` وينادي القدرة بالمستخدم أو `null`. قدرات المسار لا تستقبل جسم المقال. لذلك التصميم: فعل عام على الحارس، فعل السجل بعد `find`. `authorize` لا يكتب SQL. بعد عودتها تكمل `prisma.post.update`.

الكاش على URL قبل المعالج، مثل مفتاح بلا مستخدم، قد يخدم مقال Ada لـ Basil. الصفحة تحذّر من ذلك على المسارات التي تفحص السجل. لا تضع كاش استجابة عامًا على `GET /posts/:id` إن كانت المسودة مشروطة.

## مثال عملي

`PostPolicy` مزوّد في `PostsModule`. إنشاء المقال `@Can(..., "create")`. تعديل العنوان يحمّل الصف ثم `authorize(..., "update", user, post)`. اختبار السياسة `new PostPolicy()` من غير خادم، كما تقترح الصفحة لاستدعاء القدرة مباشرة، مع ملاحظة أن النداء المباشر يتخطى `before` إن أضفته لاحقًا.

اسم `update` في المزخرف على مسار بلا مقال يفشل الإقلاع. هذا أفضل من رفض صامت لكل المستخدمين.

## أخطاء شائعة

- `AuthorizationModule` قبل `AuthenticationModule`.
- `@Can` على قدرة تحتاج المقال.
- اعتبار أي قيمة صادقة سماحًا. الصفحة تشترط `true`.
- سياسة تحدّث Prisma.
- كاش عام على مسار يفرّق المسودات.

## متى نستخدمه؟

لمسار الدورة بعد المصادقة، لكل فعل له قاعدة أوضح من «أي مستخدم داخل».

## متى لا نستخدمه؟

قبل وجود قاعدة. `@Can` على قدرة ترجع `true` دائمًا ضوضاء. ولا تستبدل الحزمة بسلسلة شروط في المتحكم لأن المزخرف «طويل».

## مقارنة

| قدرة بلا صف | قدرة بصف |
| --- | --- |
| `@Can` | `authorize` في الخدمة |
| `create(user)` | `update(user, post)` |
| قبل المعالج | بعد `find` |

## تمارين

### تمرين سهل

أي قيمة غير `true` تسمح؟

<details>
<summary>الحل</summary>

لا شيء. حتى القيمة الصادقة غير `true` تمنع، حسب صفحة الحزمة.

</details>

### تمرين متوسط

لماذا يفشل الإقلاع إن سجّلت التفويض قبل المصادقة؟

<details>
<summary>الحل</summary>

حارس التفويض سيعمل قبل أن يُملأ المستخدم، فيرى ضيفًا. الحزمة تكتشف ترتيب الحارسين وترفض الإقلاع.

</details>

### تمرين متقدم

تريد `@Can` على تعديل المقال. لماذا ترفض الصفحة ذلك، وما البديل؟

<details>
<summary>الحل</summary>

الحارس لا يملك الصف، والقدرة `update` تحتاج المقال. البديل تحميل المقال في الخدمة ثم `authorizationService.authorize(PostPolicy, "update", user, post)`.

</details>

## أسئلة مراجعة

- أين يُستدعى `create`، وأين `update`؟
- ماذا يرمي `authorize` عند المنع؟
- كيف تُكتشف السياسة؟

## الخلاصة

`@nestjs/authorization` يضع السياسات في أصناف ترجع `true` للسماح. المسار يحمي الفعل العام، والخدمة تحمي الفعل على صف بعينه، بعد حارس المصادقة.

## ماذا بعد؟

[CASL ومتى يناسب](./06-casl.md) خيار خارج الحزمة، لا مسار الدورة.

## المصادر الرسمية

- NestJS — Authorization: https://docs.nestjs.com/security/authorization
