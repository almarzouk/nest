# AuthGuard

## الهدف من هذا الدرس

تطبيق `AuthGuard` من `@nestjs/passport` على مسار وعلى مجموعة مسارات، ومعرفة الفرق بينه وبين الحارس العام في `@nestjs/authentication`.

## المشكلة التي يحلها

فحص `req.user` في أول سطر من كل معالج يُنسى، ويعمل بعد أن بدأ المتحكم عملًا جانبيًا. كتابة الاسم `'jwt'` في عشر ملفات تكسرها عند تغيير الاسم. وضع الحارس المحلي على تطبيق كامل يطالب بكلمة المرور في كل `GET`.

## الفكرة ببساطة

`AuthGuard('jwt')` حارس Nest يشغّل استراتيجية Passport بهذا الاسم قبل المعالج. صنف `JwtAuthGuard` يخفي النص. تضعه بـ `@UseGuards` على المتحكم أو المسار. الحارس العام عبر `APP_GUARD` ممكن للوصفة إن استثنيت الدخول. حارس حزمة Security مختلف: هو العام الافتراضي مع `@Public()`.

## التعريف التقني

```ts
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

```ts
@Controller("posts")
@UseGuards(JwtAuthGuard)
export class PostsController {}
```

مسار الدخول يبقى على `LocalAuthGuard` فقط، خارج هذا الصنف.

تسجيل عام لوصفة Passport، إن اخترتها:

```ts
{
  provide: APP_GUARD,
  useClass: JwtAuthGuard,
}
```

عندها علّم `POST /auth/login` و `POST /auth/register` باستثناء. Passport لا يوفّر `@Public()` من الحزمة نفسها. الاستثناء المعتاد في الوصفات وسم بيانات وحارس يغلف `AuthGuard` ويقرأ `Reflector`، وهو نمط فصل الحراس في الدورة، لا رمز `@Public()` الخاص بـ `@nestjs/authentication`.

`@nestjs/authentication` يسجّل حارسه وحده عند `forRoot`. لا تضف `APP_GUARD` من `JwtAuthGuard` فوقه.

الحارس يعيد `false` أو يرمي فيصبح `401` قبل الخدمة. لا يقرر الدور. الدور `403` في فصل التفويض.

ترتيب الدورة: الحارس بعد الأنبوب أم قبله؟ في Nest الحارس قبل الأنابيب. طلب بلا رمز لا يصل إلى فحص DTO. هذا متوقع.

## Mental Model

```text
مسار محمي
  JwtAuthGuard
    استراتيجية jwt
    فشل -> 401
    نجاح -> req.user ثم المعالج
مسار login
  LocalAuthGuard فقط
```

## أبسط مثال

```ts
import { Body, Controller, Post, Req, UseGuards } from "@nestjs/common";
import { CreatePostDto } from "./create-post.dto.js";
import { PostsService } from "./posts.service.js";
import { JwtAuthGuard } from "../auth/jwt-auth.guard.js";

@Controller("posts")
export class PostsController {
  constructor(private readonly posts: PostsService) {}

  @Post()
  @UseGuards(JwtAuthGuard)
  create(
    @Req() req: { user: { userId: number } },
    @Body() dto: CreatePostDto,
  ) {
    return this.posts.create(req.user.userId, dto);
  }
}
```

## شرح المثال

الحارس على الإنشاء فقط في هذا المقتطف. `GET /posts` العام يمكن أن يبقى بلا حارس في الوصفة إن كان الكتالوج مفتوحًا. في حزمة Security العكس: العام مغلق وتفتح الكتالوج بـ `@Public()`. اعرف أي افتراض يعمل في المشروع الذي تقرأه.

`userId` ناتج `validate` في استراتيجية JWT، لا `id` بالضرورة. طابق الاسم الذي تُرجعه `validate`.

## ماذا يحدث داخليًا؟

`AuthGuard` ينفّذ `canActivate`. Passport يشغّل الاستراتيجية بشكل غير متزامن. النجاح يكمل إلى المعترضات والأنبوب ثم المعالج، حسب ترتيب Nest. الفشل استثناء غير مصرح. المعالج لا يلتقطه إن لم تكتب مرشحًا خاصًا. جسم `401` يأتي من طبقة الاستثناءات.

حارس على الصنف يحمي كل الطرق فيه. طريقة واحدة تُستثنى بحارس آخر أو بنقل الطريقة إلى متحكم بلا الحارس. لا يوجد في `AuthGuard('jwt')` وضع «اختياري» جاهز كـ `@Authenticate({ optional: true })`.

## مثال عملي

مكتبة على الوصفة: `PostsController` كله تحت `JwtAuthGuard` إلا إن فصلت متحكم القراءة العامة. اختبار e2e يوقّع رمزًا تجريبيًا بالمفتاح نفسه ويرسل `Bearer`. اختبار بلا رأس يتوقع `401` ولا يتوقع استدعاء `prisma.post.create`. اختبار كلمة المرور يضرب `POST /auth/login` لا `POST /posts`.

## أخطاء شائعة

- `APP_GUARD` بـ `LocalAuthGuard`.
- الجمع بين حارس Passport العام و `@Public()` من الحزمة الأخرى كأنهما نظام واحد.
- حماية `login` بـ `JwtAuthGuard` فلا يستطيع أحد الدخول.
- قراءة `req.user` من غير حارس على مسار عام.

## متى نستخدمه؟

حين تكون استراتيجية `jwt` هي بوابة المسارات في مشروع Passport.

## متى لا نستخدمه؟

على مشروع يعتمد حارس `AuthenticationModule`. هناك `@Public()` و `@CurrentUser()` هما واجهة الإغلاق والفتح.

## مقارنة

| `JwtAuthGuard` | حارس `@nestjs/authentication` |
| --- | --- |
| يشغّل Passport | يشغّل مزودات السجل |
| تغطيه حيث تضعه | عام بعد `forRoot` |
| الاستثناء نمط `Reflector` إن عمّمته | `@Public()` |
| `req.user` | `@CurrentUser()` |

## تمارين

### تمرين سهل

أي حارس على `POST /auth/login` في الوصفة، وأي حارس على `POST /posts`؟

<details>
<summary>الحل</summary>

`LocalAuthGuard` للدخول، و `JwtAuthGuard` لإنشاء المقال.

</details>

### تمرين متوسط

لماذا الحارس العام بـ `LocalAuthGuard` يعطل التطبيق؟

<details>
<summary>الحل</summary>

كل طلب سيُطالب بجسم `email` و `password` عبر الاستراتيجية المحلية، ومنها `GET` الذي لا يحمل كلمة مرور.

</details>

### تمرين متقدم

تريد كتالوج `GET /posts` مفتوحًا و `POST /posts` مغلقًا في الوصفة. أين تضع `@UseGuards`؟

<details>
<summary>الحل</summary>

على طريقة `create` فقط، أو على المتحكم ثم تفصل القراءة في متحكم بلا الحارس. لا تستخدم الحارس العام لـ JWT ثم تتوقع أن القراءة مفتوحة من غير استثناء صريح.

</details>

## أسئلة مراجعة

- ماذا يغلف `JwtAuthGuard`؟
- هل هو حارس الأدوار؟
- كيف يختلف الافتراض عن حارس الحزمة الرسمية؟

## الخلاصة

`AuthGuard` بوابة استراتيجية Passport. استخدم `jwt` للمسارات المحمية و `local` للدخول فقط. لا تخلطه مع حارس `@nestjs/authentication` على السلسلة نفسها.

## ماذا بعد؟

[متى الحزمة الرسمية ومتى Passport](./12-which-approach.md) قرار المسار.

## المصادر الرسمية

- NestJS — Passport: https://docs.nestjs.com/recipes/passport
- NestJS — Guards: https://docs.nestjs.com/guards
- NestJS — Authentication: https://docs.nestjs.com/security/authentication
