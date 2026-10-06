# JWT Strategy

## الهدف من هذا الدرس

حماية المسارات برمز حامل كما في وصفة Passport: إصدار بتوقيع `@nestjs/jwt` وفحص بـ `passport-jwt`.

## المشكلة التي يحلها

بعد `POST /auth/login` يملك العميل سلسلة. من غير استراتيجية ثانية، كل طلب لاحق لا يعرف كيف يقرأ `Authorization`. الوثوق بالسلسلة من غير فحص التوقيع و `exp` يعيد قبول أي JSON.

## الفكرة ببساطة

`JwtModule.register` يعرف السر وعمر التوقيع. الدخول المحلي، بعد نجاح الحارس، يستدعي `sign` ويضع `sub` معرّف المستخدم. `JwtStrategy` تقرأ الحامل، وPassport يتحقق من التوقيع والانتهاء، ثم `validate` ترجع ما سيصبح `req.user`. `JwtAuthGuard` يغلّف `AuthGuard('jwt')` على المسارات المحمية.

## التعريف التقني

من الوصفة:

```ts
JwtModule.register({
  secret: jwtConstants.secret,
  signOptions: { expiresIn: "60s" },
});
```

`expiresIn` هنا بصيغة `@nestjs/jwt`، والمثال الرسمي قصير `60s` للعرض. في تمرين حقيقي اختر عمرًا قصيرًا مفهومًا، والسر من `ConfigService` عبر `registerAsync` لا من ملف `constants.ts` مرفوع. لا تنسخ سر المثال.

```ts
import { ExtractJwt, Strategy } from "passport-jwt";
import { PassportStrategy } from "@nestjs/passport";
import { Injectable } from "@nestjs/common";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  validate(payload: { sub: number; email: string }) {
    return { userId: payload.sub, email: payload.email };
  }
}
```

`ignoreExpiration: false` هو الافتراضي الصريح في الصفحة: الرمز المنتهي `401`. `validate` تستلم الحمولة بعد نجاح التوقيع، لا قبلها. ما تُرجعه يصبح `req.user`. يمكنك جلب المستخدم من القاعدة هنا إن أردت إبطالًا أقوى من عمر الرمز.

الإصدار في خدمة الدخول:

```ts
return {
  access_token: await this.jwt.signAsync({ sub: user.id, email: user.email }),
};
```

هذه الوصفة في شكلها الأساسي بلا رمز تحديث دوار. التحديث الدوار والعائلة موجودان في `@nestjs/authentication` لا في هذا المثال. لا تخترع `TokenService` داخل استراتيجية Passport.

## Mental Model

```text
login محلي ناجح -> sign({ sub }) -> access_token
طلب تالٍ
  Authorization: Bearer
  passport-jwt يفحص التوقيع و exp
  validate(payload) -> req.user
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

```ts
@UseGuards(JwtAuthGuard)
@Get("me")
me(@Req() req: { user: { userId: number; email: string } }) {
  return req.user;
}
```

## شرح المثال

اسم `jwt` هو الاسم الافتراضي لاستراتيجية `passport-jwt` عند `PassportStrategy(Strategy)`. الحارس المحلي يبقى على `login` فقط. هذا الحارس على `me` و `POST /posts`. المعالج لا يقرأ الرأس بنفسه.

الحمولة صغيرة: معرّف وبريد. لا تضع التجزئة ولا كلمة المرور. الدور إن وضعته قد يقدُم، لذلك جلب الصف داخل `validate` أفضل حين يتغير الدور.

## ماذا يحدث داخليًا؟

`ExtractJwt.fromAuthHeaderAsBearerToken` يأخذ الجزء بعد `Bearer `. Passport يتحقق بالمفتاح نفسه الذي وقّع. اختلاف السر أو انتهاء العمر يوقف الطلب قبل `validate`. `validate` لا تعيد فحص التوقيع. دورها قرار إضافي: بناء كائن المستخدم أو رفض معرّف محذوف.

العميل يخزّن `access_token` ويعيده. السرقة قبل `exp` تنتحل المستخدم. الوصفة الأساسية لا تسحب الرمز. إن احتجت سحبًا، إما عمر قصير جدًا أو مخزن إبطال تفحصه في `validate`، أو تنتقل إلى نموذج التحديث في حزمة Security.

## مثال عملي

`POST /auth/login` يرجع الرمز. `GET /posts` بلا رأس `401`. بالرأس يعيد الصفحة. `POST /posts` يقرأ `req.user.userId` كمؤلف ويتجاهل أي `authorId` في الجسم. بعد `60s` في مثال التوثيق، الرمز نفسه `401` بسبب `ignoreExpiration: false`.

أبقِ السر في البيئة. مثال `jwtConstants.secret` في المقالات التعليمية ليس إعداد إنتاج.

## أخطاء شائعة

- سر مختلف بين `JwtModule.register` و `secretOrKey`.
- `ignoreExpiration: true` في الإنتاج.
- حمولة فيها `passwordHash`.
- استخدام هذا الحارس على مسار `login` الذي يرسل كلمة مرور لا حاملًا.
- ظن أن الوصفة تتضمن `refresh` دوارًا كما في `TokenService`.

## متى نستخدمه؟

داخل مشروع Passport بعد نجاح الاستراتيجية المحلية، لحماية API بالرأس `Bearer`.

## متى لا نستخدمه؟

إن كان `JwtBearerProvider` مسجّلًا أصلًا لفحص الحامل. حارسان على الرأس نفسه يربكان مصدر `401`.

## مقارنة

| JWT في الوصفة | وصول `@nestjs/authentication` |
| --- | --- |
| `@nestjs/jwt` بالثواني | `accessToken.ttl` بنص أو مللي ثانية |
| `AuthGuard('jwt')` | `JwtBearerProvider` |
| تحديث ليس في المثال الأساسي | `TokenService.refresh` دوار |
| `req.user` | `@CurrentUser()` |

## تمارين

### تمرين سهل

متى تُستدعى `validate` في `JwtStrategy`، قبل فحص التوقيع أم بعده؟

<details>
<summary>الحل</summary>

بعد نجاح التوقيع وفك الحمولة. Passport لا يستدعيها لرمز تالف.

</details>

### تمرين متوسط

لماذا يجب أن يتطابق السر في `sign` و `secretOrKey`؟

<details>
<summary>الحل</summary>

الفحص يعيد حساب التوقيع بالمفتاح. مفتاح آخر يرفض رمزًا أصدره التطبيق نفسه.

</details>

### تمرين متقدم

المستخدم غيّر بريده وما زال الرمز يحوي البريد القديم. أين تُحدّث القيمة التي يراها `GET /me`؟

<details>
<summary>الحل</summary>

في `validate` تجلب الصف الحالي بالمعرّف `sub` وتُرجع بريده الجديد، بدل نسخ `payload.email` كما هو.

</details>

## أسئلة مراجعة

- أي رأس يقرأه `fromAuthHeaderAsBearerToken`؟
- ماذا يفعل `ignoreExpiration: false`؟
- هل هذه الوصفة هي `TokenService`؟

## الخلاصة

استراتيجية JWT في Passport تفحص الحامل بعد توقيع من `@nestjs/jwt`. `validate` تبني `req.user` من حمولة صحيحة. التحديث الدوار مسار الحزمة الأخرى.

## ماذا بعد؟

[AuthGuard](./11-auth-guard.md) يضع الحارس على المسار وعلى التطبيق بحذر.

## المصادر الرسمية

- NestJS — Passport: https://docs.nestjs.com/recipes/passport
- NestJS — Guards: https://docs.nestjs.com/guards
