# قراءة المستخدم الحالي

## الهدف من هذا الدرس

قراءة الشخص الذي ثبّته مزوّد المصادقة داخل المتحكم أو الخدمة، وربط `POST /posts` به.

## المشكلة التي يحلها

بعد الحارس ما زال المتحكم يطلب `authorId` من JSON. تمرير `req.user` عبر كل دالة يلوّث التواقيع ويكسر الاختبار. نسيان معامل في مسار جديد يكتب المقال باسم يأتي من العميل.

## الفكرة ببساطة

`@CurrentUser()` يحقن ما رجعه `validate`. `@CurrentUser("id")` يحقن خاصية واحدة. الخدمة التي لا تريد معاملًا تقرأ `AuthenticationContext`. المنشور يُنشأ لصاحب السياق، والجسم يبقى عنوانًا ونصًا.

## التعريف التقني

من صفحة المصادقة:

```ts
import { Controller, Get } from "@nestjs/common";
import { CurrentUser } from "@nestjs/authentication";

@Controller("me")
export class MeController {
  @Get()
  me(@CurrentUser() user: User) {
    return user;
  }
}
```

`@CurrentUser("id")` للخاصية. `@CurrentSession()` لسجل الجلسة عند مزوّد الكعكة. توسيع النوع اختياري عبر `declare module "@nestjs/authentication"` وواجهة `AuthenticationTypes` حتى يقبل `@CurrentUser("key")` مفاتيح `User` فقط. التوسعة لا تغيّر وسم المعامل نفسه: تكتب `user: User` بيدك. نوع واحد للتطبيق كله.

`AuthenticationContext` للخدمة. مثال الصفحة: الطلبات تقرأ العميل من السياق حتى لا يمرّر المتحكم المستخدم في كل نداء. إن كان المسار `@Public()` والسياق فارغًا، لا تستخدم الحقل كأنه شخص.

`@Authenticate({ optional: true })` للمسار الذي يعمل لضيف ومستخدم، مثل كتالوج يرى مسودات للموظف فقط لاحقًا في التفويض.

على `CreatePostDto` احذف `authorId` بعد هذه الخطوة.

## Mental Model

```text
حارس -> validate رجعت User
        |
 @CurrentUser() في المتحكم
 AuthenticationContext في الخدمة
        |
 PostsService.create(title, body) تقرأ id من السياق
```

## أبسط مثال

```ts
import { Body, Controller, Post } from "@nestjs/common";
import { CurrentUser } from "@nestjs/authentication";
import { CreatePostDto } from "./create-post.dto.js";
import { PostsService } from "./posts.service.js";
import type { PublicUser } from "../users/public-user.js";

@Controller("posts")
export class PostsController {
  constructor(private readonly posts: PostsService) {}

  @Post()
  create(@CurrentUser() user: PublicUser, @Body() dto: CreatePostDto) {
    return this.posts.create(user.id, dto);
  }
}
```

## شرح المثال

`CreatePostDto` فيه `title` و `body` فقط. المعرّف وسيط ثانٍ من نوع هويتك، نص أو عدد حسب الجدول. اختبار المتحكم يمرّر مستخدمًا مزيفًا عبر طبقة HTTP التي تضع الدليل، أو يختبر الخدمة بتمرير `id` مباشرة إن فصلت الدالة عن السياق.

إن استخدمت السياق داخل الخدمة، اختبار الوحدة يحتاج أن يملأ السياق بالطريقة التي تعرضها الحزمة، وإلا فالتوقيع الصريح `create(authorId, dto)` أسهل في الاختبار والمعنى واحد: المؤلف لا يأتي من DTO.

## ماذا يحدث داخليًا؟

المزوّد وضع المستخدم على الطلب بعد نجاح الدليل. المزخرف يقرأ تلك القيمة عند استدعاء المعالج. إن وصلت إلى معالج غير عام بلا مستخدم، الحارس سبق أن أوقف الطلب بـ `401`، فالمعامل في المسار المغلق موجود. على مسار عام، المعامل قد يكون فارغًا: لا تفترض `user.id` من غير فحص.

`select` في Prisma ما زال يمنع `passwordHash`. `@CurrentUser()` يجب أن يكون الشخص العام الذي رجعه `validate`، لا صف القاعدة الكامل. إن أعاد `validate` الصف مع التجزئة، فـ `GET /me` يسرّبها. أصلح `validate` لا القالب فقط.

## مثال عملي

`GET /me` يعيد `{ id, email, name }` لمن حمل الكعكة. `POST /posts` بعنوان «جلسة» ينشئ صفًا بـ `authorId` نفسه. طلب ثانٍ يحاول إرسال `authorId` في JSON فيرفضه الأنبوب كحقل زائد. مستخدم آخر يدخل ويكتب مقالًا بمعرّفه هو. قائمة `GET /posts?authorId=` للتصفية ما زالت استعلام قراءة، وتختلف عن حقل الإنشاء: التصفية تسأل «مقالات من؟» والإنشاء يقرر «باسم من».

## أخطاء شائعة

- إرجاع صف Prisma كامل من `validate`.
- الإبقاء على `authorId` في DTO «للتوافق» من غير تجاهله.
- `@CurrentUser()` على مسار `@Public()` ثم الوصول إلى `id` بلا فحص.
- تمرير `request` إلى `PostsService`.

## متى نستخدمه؟

في كل كتابة يجب أن تُنسب إلى الشخص الذي أثبت الحارس هويته.

## متى لا نستخدمه؟

في سكربت داخلي بلا طلب HTTP. هناك المعرّف معامل صريح من موجّه المهمة، لا مزخرف طلب.

## مقارنة

| `authorId` في الجسم | `@CurrentUser()` |
| --- | --- |
| يختاره العميل | يثبته المزوّد |
| مناسب قبل هذا الفصل | مصدر الإنشاء بعد الدخول |
| يُحذف من DTO الكتابة | يبقى في استعلام التصفية إن لزم |

## تمارين

### تمرين سهل

ماذا يحقن `@CurrentUser("id")`؟

<details>
<summary>الحل</summary>

خاصية `id` من المستخدم الذي رجعه `validate`، لا الجسم.

</details>

### تمرين متوسط

لماذا فشل `GET /me` بتسريب التجزئة رغم `select` في `UsersService.findOne`؟

<details>
<summary>الحل</summary>

`validate` أعادت صفًا آخر فيه `passwordHash`. المزخرف يحقن ما رجعه المزوّد. ضيّق قيمة `validate` إلى الشخص العام.

</details>

### تمرين متقدم

مسار الكتالوج `@Public()` ويعرض زر «مسوداتي» إن وُجد مستخدم. أي أداة تقرأ الضيف والمستخدم من غير إجبار `401`؟

<details>
<summary>الحل</summary>

`@Authenticate({ optional: true })` ثم `@CurrentUser()` الذي قد يكون فارغًا، والخدمة تتفرع على الغياب. لا تضع المسار خلف الحارس العام ثم تستثنيه بالكامل فتفقد المستخدم إن وُجدت كعكة.

</details>

## أسئلة مراجعة

- من يملأ قيمة `@CurrentUser()`؟
- لماذا حُذف `authorId` من إنشاء المقال؟
- أين تقرأ الخدمة الهوية إن لم يمرّرها المتحكم؟

## الخلاصة

المستخدم الحالي ناتج `validate`. المتحكم يقرأه بمزخرف، والكتابة تنسبه إليه. الجسم لم يعد يختار المؤلف.

## ماذا بعد؟

[Bearer Tokens و Refresh Tokens](./07-bearer-and-refresh-tokens.md) للجوال الذي لا يرسل كعكة.

## المصادر الرسمية

- NestJS — Authentication: https://docs.nestjs.com/security/authentication
