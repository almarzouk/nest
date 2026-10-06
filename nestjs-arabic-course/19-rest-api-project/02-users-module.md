# Users

## الهدف من هذا الدرس

بناء `UsersModule`: إنشاء مستخدم بالبريد والاسم، وقراءته بالمعرّف، من غير إرجاع تجزئة كلمة المرور.

## المشكلة التي يحلها

منشور بلا مؤلف ثابت يربط `authorId` بقيمة لا تملك صفًا. إن عاش المستخدم داخل `PostsService`، حذف البريد أو فحص التكرار يتناثر. المورد `users` هو جدول `User` كما رُسم في الحدود.

## الفكرة ببساطة

`UsersService` تنشئ وتجلب. `UsersController` يترجم HTTP. البريد الفريد قيد في PostgreSQL. كلمة المرور في هذا الفصل تُخزَّن مؤقتًا كقيمة لا تُعاد، والفصل `20` يستبدل التخزين الساذج بـ `PasswordHasher`. لا تُبقِ كلمة مرور واضحة في عمود إنتاج. العمود اسمه `passwordHash` من الآن حتى لا يعاد تسمية الترحيل مرتين، والقيمة حتى فصل المصادقة ليست تجزئة حقيقية: لا تنشر الوحدة بهذا الشكل.

الأسلم في التمرين المحلي: ارفض تشغيل المسار إن لم يكن `NODE_ENV=development`، أو خزّن ناتج تجزئة لاحقًا. الدرس يركز على حدود الوحدة، ويذكر أن القيمة الواضحة ممنوعة خارج جهازك.

## التعريف التقني

```ts
@Controller("users")
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.users.create(dto);
  }

  @Get(":id")
  findOne(@Param("id", ParseIntPipe) id: number) {
    return this.users.findOne(id);
  }
}
```

`CreateUserDto` فيه `email` و `name` و `password` بوسوم class-validator. الخدمة تكتب `passwordHash: dto.password` فقط في فرع تطوير معزول، وتستبدله في فصل المصادقة باستدعاء `hash`. الاستجابة:

```ts
select: { id: true, email: true, name: true }
```

`findUnique` إن رجعت `null` ترمي الخدمة `NotFoundException`. تكرار البريد خطأ Prisma يُترجم إلى `ConflictException`.

الوحدة:

```ts
@Module({
  imports: [PrismaModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

`exports` حتى تتأكد `PostsModule` من المؤلف من غير فتح `PrismaService` على جدول المستخدم إن رغبت بحد أوضح. كلا الأسلوبين مقبول: فحص المؤلف عبر `UsersService.findOne` يصدّر قاعدة المستخدم من وحدته.

## Mental Model

```text
POST /users  -> DTO -> UsersService.create -> prisma.user.create
GET /users/4 -> ParseIntPipe -> findOne -> 404 أو { id, email, name }
```

## أبسط مثال

```ts
import { ConflictException, Injectable, NotFoundException } from "@nestjs/common";
import { Prisma } from "../generated/prisma/client.js";
import { PrismaService } from "../prisma/prisma.service.js";

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async create(input: { email: string; name: string; passwordHash: string }) {
    try {
      return await this.prisma.user.create({
        data: input,
        select: { id: true, email: true, name: true },
      });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002") {
        throw new ConflictException("Email already registered");
      }
      throw error;
    }
  }

  async findOne(id: number) {
    const user = await this.prisma.user.findUnique({
      where: { id },
      select: { id: true, email: true, name: true },
    });
    if (!user) {
      throw new NotFoundException("User not found");
    }
    return user;
  }
}
```

## شرح المثال

الخدمة لا تستقبل كلمة المرور باسم حقل الاستجابة. المتحكم يمرّر `passwordHash` من DTO في هذا الفصل التعليمي، وستتغيّر هذه الوسيطة حين يدخل المجزّئ. `select` يمنع العمود السري حتى لو أضفت حقولًا لاحقًا ونسيت. رمز `P2002` قيد الفرادة في Prisma. ترجمته تخفي تفاصيل المشغّل.

## ماذا يحدث داخليًا؟

`ParseIntPipe` يرفض `id` غير العددي بـ `400` قبل الخدمة. `create` يرسل `INSERT` ويرجع الأعمدة المختارة. مرشح الاستثناءات يحوّل `NotFoundException` إلى جسم فيه `404`. لا يوجد حارس. المسار مفتوح داخل حدود هذا الفصل.

## مثال عملي

أنشئ مستخدمًا `ada@example.com`. أعِد الطلب نفسه وتوقع `409`. اطلب `GET /users/999` وتوقع `404`. اطلب `GET /users/1` وتأكد أن JSON فيه `id` و `email` و `name` فقط. اختبار الوحدة يزيف `prisma.user.create` ولا يتصل بقاعدة.

## أخطاء شائعة

- إرجاع كائن Prisma كاملًا.
- فحص تكرار البريد فقط في الذاكرة.
- جعل `PostsService` تكتب في `prisma.user`.
- الإبقاء على كلمة مرور واضحة بعد إنهاء فصل المصادقة.

## متى نستخدمه؟

 حين احتاج منشور أو تعليق لاحقًا إلى مؤلف له صف حقيقي.

## متى لا نستخدمه؟

كنظام حسابات إنتاج قبل التجزئة وقبل إغلاق `POST /users` خلف سياسة التسجيل.

## مقارنة

| `select` في الخدمة | إرجاع النموذج كاملًا |
| --- | --- |
| العقد يسمي الحقول | العمود السري قد يخرج |
| يتغير بتغيير العقد | يتغير بتغيير الجدول |

## تمارين

### تمرين سهل

أي استثناء لتكرار البريد، وأي استثناء لمستخدم غائب؟

<details>
<summary>الحل</summary>

`ConflictException` للتكرار، و `NotFoundException` للغياب.

</details>

### تمرين متوسط

لماذا تُصدَّر `UsersService` من الوحدة؟

<details>
<summary>الحل</summary>

حتى تفحص `PostsModule` وجود المؤلف عبر قاعدة المستخدمين لا عبر SQL منسوخ.

</details>

### تمرين متقدم

طلبان ينشئان البريد نفسه في اللحظة نفسها وتجاوزا فحصًا مبكرًا. ماذا يبقى حارسًا؟

<details>
<summary>الحل</summary>

`@unique` ورمز `P2002` الذي يتحوّل إلى `409`. الفحص المبكر يحسّن الرسالة الشائعة ولا يستبدل القيد.

</details>

## أسئلة مراجعة

- ماذا يعيد `GET /users/:id`؟
- لماذا `select`؟
- من يملك كتابة جدول المستخدم؟

## الخلاصة

مورد المستخدم وحدة مستقلة، العقد بلا سر، والتكرار والغياب استثناءات HTTP واضحة.

## ماذا بعد؟

[Posts](./03-posts-module.md) يربط المنشور بمؤلف من هذه الوحدة.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- NestJS — Prisma: https://docs.nestjs.com/data/prisma
