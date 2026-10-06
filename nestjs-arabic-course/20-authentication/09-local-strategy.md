# Local Strategy

## الهدف من هذا الدرس

تنفيذ دخول البريد وكلمة المرور كما في وصفة Passport: `LocalStrategy` و `AuthGuard('local')` ومسار `POST /auth/login`.

## المشكلة التي يحلها

استدعاء `passport.authenticate('local')` يدويًا داخل كل إطار يكرر ربط Express. كلمة المرور تُقارن في المتحكم بجانب `res.json`. فشل المستخدم الغائب برسالة غير فشل الكلمة الخاطئة.

## الفكرة ببساطة

الاستراتيجية تقرأ الجسم، وتستدعي خدمة تتحقق من التجزئة، وترجع الشخص أو ترمي `401`. الحارس على مسار الدخول فقط يشغّل هذه الاستراتيجية. المسار نفسه لا يحمي بقية التطبيق. حماية البقية استراتيجية JWT في الدرس التالي، أو حزمة Security إن كان المشروع عليها أصلًا.

## التعريف التقني

من الوصفة، مع بريد المكتبة:

```ts
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly auth: AuthService) {
    super({ usernameField: "email" });
  }

  async validate(email: string, password: string) {
    const user = await this.auth.validateUser(email, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

توقيع `validate(username, password)` هو ما يتوقعه `passport-local`. المعامل الأول هنا بريد بسبب الخيار.

```ts
@Injectable()
export class LocalAuthGuard extends AuthGuard("local") {}
```

```ts
@Post("login")
@UseGuards(LocalAuthGuard)
login(@Req() req: { user: PublicUser }) {
  return this.auth.issueSessionOrToken(req.user);
}
```

الصفحة تستخدم `@Request()` من `@nestjs/common` أيضًا. بعد الحارس، `req.user` هو ناتج `validate`. إصدار JWT في الوصفة يتم هنا باستدعاء `@nestjs/jwt`، لا داخل `validate`. `validate` تثبت فقط.

`AuthService.validateUser` تجلب التجزئة، وتنادي `verify`، وترجع `{ id, email, name }` أو `null`. رسالة الرفض واحدة.

سجّل `LocalStrategy` في `providers` مع `PassportModule` و `UsersModule`.

هذه الوصفة لا تضع كعكة `__Host-sid` إلا إن كتبت ذلك بعد النجاح. مثال التوثيق الشهير يصدر `access_token` بعد الدخول المحلي.

## Mental Model

```text
POST /auth/login
  LocalAuthGuard
    passport-local يقرأ email و password
    validate -> PublicUser أو 401
  المعالج يرى req.user ويصدر دليلًا
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { PasswordHasher } from "@nestjs/authentication";
import { UsersService } from "../users/users.service.js";

@Injectable()
export class AuthService {
  constructor(
    private readonly users: UsersService,
    private readonly hasher: PasswordHasher,
  ) {}

  async validateUser(email: string, password: string) {
    const found = await this.users.findCredentials(email);
    const ok = await this.hasher.verify(password, found?.passwordHash);
    if (!ok || !found?.passwordHash) {
      return null;
    }
    return found.user;
  }
}
```

## شرح المثال

الخدمة صالحة حتى لو استبدلت Passport لاحقًا بـ `CredentialsService` في الحزمة الرسمية: المقارنة في مكان واحد. `findCredentials` ترجع التجزئة للدخول فقط، لا لمسار `GET /users/:id`.

حارس المحلية لا يُوضع على `GET /posts`. ذلك الحارس يطالب بجسم فيه كلمة مرور في كل طلب. حماية القراءة استراتيجية أخرى أو جلسة.

## ماذا يحدث داخليًا؟

Passport يحلّل الجسم قبل المعالج. إن غاب `email` تفشل الاستراتيجية. `validate` تعمل داخل طلب الدخول فقط. النجاح يمرّر التحكم إلى `login`، وهناك توقّع الوصفة إصدار رمز. لا يُنشأ `req.user` على الطلبات التالية إلا إذا حملت تلك الطلبات دليلًا تفحصه استراتيجية أخرى.

`@UseGuards(LocalAuthGuard)` على الدخول يعني أن المعالج لا يُستدعى عند الفشل، فلا تسجّل نجاحًا كاذبًا.

## مثال عملي

عميل المكتبة يرسل `{ "email": "ada@example.com", "password": "correct-horse" }` إلى `POST /auth/login`. النجاح يرجع `{ "access_token": "..." }` في شكل الوصفة. كلمة خاطئة `401` من غير جسم المستخدم. ثم يطلب `GET /posts` بالرمز، وهذا لن يمر على `LocalAuthGuard`. الدرس التالي يضع `JwtAuthGuard` على القراءة.

إن كان مشروع الدورة على `@nestjs/authentication`، أبقِ هذا الملف مثال قراءة لا تستورده في `AppModule` مع `SessionAuth` على المسار نفسه.

## أخطاء شائعة

- حارس `local` على كل المسارات.
- مقارنة كلمة المرور داخل `login` بعد أن أعادت `validate` التجزئة إلى العميل.
- `username` في الجسم بينما الخيار يقول `email`.
- نسيان تسجيل الصنف كمزوّد.

## متى نستخدمه؟

في مشروع يتبع وصفة Passport للخطوة الأولى: إثبات الكلمة ثم إصدار دليل في المعالج.

## متى لا نستخدمه؟

كحارس عام للتطبيق. ولا بجانب `SignInService` على نفس مسار الدخول من غير إزالة أحدهما.

## مقارنة

| `LocalStrategy` | `SessionAuth` في الحزمة |
| --- | --- |
| تثبت الكلمة في طلب الدخول | تقرأ كعكة في الطلبات التالية |
| `AuthGuard('local')` | حارس الحزمة العام |
| `req.user` بعد Passport | `@CurrentUser()` بعد المزوّد |

## تمارين

### تمرين سهل

لماذا لا يُوضع `LocalAuthGuard` على `GET /me`؟

<details>
<summary>الحل</summary>

لأنه يتوقع جسم دخول بكلمة مرور. طلبات الهوية التالية تحمل رمزًا أو كعكة، لا الكلمة في كل مرة.

</details>

### تمرين متوسط

ماذا يجب أن يرجع `validate` حتى لا يسرّب العمود السري إلى `req.user`؟

<details>
<summary>الحل</summary>

الشخص العام `{ id, email, name }` من غير `passwordHash`.

</details>

### تمرين متقدم

الوصفة تصدر JWT في `login` لا في `validate`. ماذا تستفيد الاختبارات من هذا الفصل؟

<details>
<summary>الحل</summary>

`validateUser` تُختبر بكلمة صحيحة وخاطئة من غير توقيع ولا HTTP. إصدار الرمز يُختبر على المعالج بعد مستخدم جاهز.

</details>

## أسئلة مراجعة

- ما توقيع `validate` في الاستراتيجية المحلية؟
- متى يُملأ `req.user`؟
- أين تُقارن التجزئة؟

## الخلاصة

Local Strategy تثبت البريد والكلمة لطلب دخول واحد وتضع شخصًا عامًا على الطلب. حماية بقية المسارات دليل آخر.

## ماذا بعد؟

[JWT Strategy](./10-jwt-strategy.md) تفحص `Bearer` في وصفة Passport.

## المصادر الرسمية

- NestJS — Passport: https://docs.nestjs.com/recipes/passport
