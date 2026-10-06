# Passport و @nestjs/passport

## الهدف من هذا الدرس

فهم وصفة Passport كما هي في Recipes: استراتيجية، دالة `validate`، و `AuthGuard`، من غير اعتبارها المسار الوحيد ولا مسارًا ملغى.

## المشكلة التي يحلها

مقالات كثيرة تبدأ بـ `passport-local` فيظن القارئ أن قسم Security الجديد ألغاها، أو يخلط `AuthGuard('local')` بحارس `@nestjs/authentication`. المشروع القائم على Passport ينكسر إن حذفته لأنه «قديم». المشروع الجديد ينسخ الوصفة كاملة ثم يفاجئه أن إصدار الرمز والتحديث الدوار ليسا هما `TokenService`.

## الفكرة ببساطة

Passport مكتبة Node شهيرة. `@nestjs/passport` يلفها كاستراتيجية قابلة للحقن. كل استراتيجية صنف يمتد `PassportStrategy` وينفّذ `validate`. الحارس `AuthGuard('name')` يشغّل الاستراتيجية المسجّلة بذلك الاسم. النجاح يضع `req.user`. الفشل يرفض الطلب. الوصفة في التوثيق تبني دخولًا محليًا ثم JWT. هذا مسار رسمي ثانٍ بجانب الحزمة التي سبقت.

## التعريف التقني

التثبيت الذي تبدأ به صفحة Passport:

```bash
npm install --save @nestjs/passport passport passport-local
npm install --save-dev @types/passport-local
```

ثم `passport-jwt` و `@nestjs/jwt` حين تصل إلى الرمز. `@nestjs/jwt` يعدّ `expiresIn` بالثواني في `signOptions`، مثل `60s` في مثال الصفحة. لا تنقل هذا الرقم إلى خيار `ttl` في `@nestjs/authentication`.

شكل الوحدة:

```ts
@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

`PassportModule` تسجيل التكامل. الاستراتيجية مزوّد. من غير وجودها في `providers` لا يعرف Passport اسم `local`.

دالة `validate` هي رد النداء الذي يتوقعه Passport. ما تُرجعه يصبح `req.user`. رمي `UnauthorizedException` فشل إثبات. الصفحة تقول إن فشل التحقق يعني مستخدمًا غير موجود أو كلمة لا تطابق، حسب الاستراتيجية.

لا تسجّل حارس `@nestjs/authentication` العام وحارس Passport على المسار نفسه من غير أن تعرف أيهما يكتب `req.user`. للدورة: افهم الوصفة على مثال منفصل، وأبقِ مشروع المنشورات على حزمة Security إن بدأت بها.

## Mental Model

```text
استراتيجية Passport
  super(options)  خيارات المكتبة
  validate(...)   تثبت المستخدم
AuthGuard("name") يشغّل الاستراتيجية
req.user          ناتج validate
```

## أبسط مثال

```ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { Strategy } from "passport-local";
import { AuthService } from "./auth.service.js";

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

## شرح المثال

`passport-local` افتراضيًا يقرأ `username` و `password`. نموذج الدورة يرسل `email`، لذلك `usernameField`. `validateUser` تستدعي `PasswordHasher.verify` أو مقارنة مشروعك، وترجع الشخص العام أو `null`. الاستراتيجية لا تكتب الكعكة بنفسها في هذه الوصفة: الدخول الناجح غالبًا يوقّع JWT في المتحكم بعد أن يملأ الحارس `req.user`.

هذا ملف الوصفة، لا مزوّد `SessionCookieProvider`.

## ماذا يحدث داخليًا؟

`AuthGuard('local')` يستدعي Passport، وPassport يقرأ الجسم، وينادي `validate`، وعند النجاح يضع المستخدم على الطلب ويكمل السلسلة نحو المعالج. عند الفشل تصل `401`. الاستراتيجية تُسجَّل بالاسم الافتراضي `local` المشتق من الحزمة. الاسم في `AuthGuard` يجب أن يطابقه. الصفحة تنصح بصنف `LocalAuthGuard extends AuthGuard('local')` حتى لا يتكرر النص السحري.

JWT استراتيجية ثانية باسم `jwt`، لا استبدال لصنف المحلية.

## مثال عملي

مشروع قديم للمكتبة فيه `POST /auth/login` مع الحارس المحلي و `GET /posts` مع حارس JWT. اقرأه بهذه الخريطة ولا تعِد كتابته في منتصف الدورة إلا بقرار آخر الدرس. مشروع جديد اتبع `@nestjs/authentication` لأن قسم Security هو ما تبني عليه الفصول التالية من التفويض الرسمي `@nestjs/authorization`، وهو يتوقع مستخدم تلك الحزمة. Passport يبقى مفهومًا لقراءة القضايا والوظائف.

## أخطاء شائعة

- القول إن Passport أُلغي في Nest 12.
- خلط `AuthGuard('jwt')` مع `JwtBearerProvider`.
- نسيان `PassportModule` أو نسيان الاستراتيجية في `providers`.
- إرجاع صف فيه `passwordHash` من `validate` فيصبح `req.user`.

## متى نستخدمه؟

لقراءة الوصفة الرسمية، ولمشروع قائم عليها، ولفريق يريد استراتيجيات Passport الجاهزة في المجتمع.

## متى لا نستخدمه؟

كطبقة ثانية على مسارات تستخدم أصلًا حارس `@nestjs/authentication`، من غير خطة لمن يملأ `req.user`.

## مقارنة

| `@nestjs/authentication` | Passport في Recipes |
| --- | --- |
| قسم Security الحالي | وصفة ما زالت موثّقة |
| مزودات تسجّل في `AuthenticationRegistry` | `PassportStrategy` و `AuthGuard` |
| جلسة وتحديث دوار في الحزمة | JWT عبر `@nestjs/jwt` في المثال |
| ليس بديلًا يلغي الآخر | ليس المسار الوحيد |

## تمارين

### تمرين سهل

ماذا تصبح قيمة رجوع `validate` على الطلب؟

<details>
<summary>الحل</summary>

`req.user`، بعد نجاح استراتيجية Passport.

</details>

### تمرين متوسط

لماذا `usernameField: "email"` في مثال المكتبة؟

<details>
<summary>الحل</summary>

`passport-local` يبحث عن `username` افتراضيًا، ونموذج الدخول يرسل `email`.

</details>

### تمرين متقدم

لماذا لا نضع `AuthenticationModule` و `AuthGuard('local')` على `POST /auth/login` معًا من غير قراءة الترتيب؟

<details>
<summary>الحل</summary>

حارسان قد يرفضان الطلب أو يكتبان مستخدمًا بطريقتين. اختر سلسلة واحدة للمسار حتى تعرف مصدر `401` ومصدر الهوية.

</details>

## أسئلة مراجعة

- هل الوصفة ملغاة؟
- ماذا يسجّل `PassportModule` مع الاستراتيجية؟
- كيف يختلف عمر `@nestjs/jwt` عن `ttl` في حزمة المصادقة؟

## الخلاصة

Passport استراتيجية تحقق وحارس بالاسم، ووصفة رسمية قائمة. حزمة Security مسار الدورة الجديد. الدروس التالية تفصل Local و JWT و `AuthGuard` داخل الوصفة فقط.

## ماذا بعد؟

[Local Strategy](./09-local-strategy.md) تفاصيل دخول البريد وكلمة المرور في Passport.

## المصادر الرسمية

- NestJS — Passport: https://docs.nestjs.com/recipes/passport
- NestJS — Authentication: https://docs.nestjs.com/security/authentication
