# Bearer Tokens و Refresh Tokens

## الهدف من هذا الدرس

إصدار رمز وصول قصير ورمز تحديث دوار عبر `TokenService`، والتحقق منهما بـ `JwtBearerProvider` بجانب جلسة الكعكة.

## المشكلة التي يحلها

تطبيق الجوال لا يدير كعكة المتصفح كما تفعل واجهة نفس الموقع. نسخ الجلسة إلى ترويسة يخترعها كل عميل يفوّت فحص التوقيع والانتهاء. رمز واحد طويل العمر لا يُسحب عند ضياع الهاتف إلا بقائمة سوداء لكل طلب.

## الفكرة ببساطة

`POST /auth/token` يثبت الكلمة ثم يستدعي `issue`. العميل يرسل الوصول في `Authorization: Bearer`. عند انتهائه يرسل التحديث إلى `POST /auth/token/refresh` فيأخذ زوجًا جديدًا. التحديث يُستخدم مرة. مزوّد `JwtAuth` يتحقق من الوصول بعد مزوّد الجلسة، فالكعكة والجوال يتعايشان.

## التعريف التقني

خيار التوقيع في `forRoot` أو `forRootAsync`:

```ts
accessToken: {
  key: process.env.JWT_SECRET!,
  issuer: "https://api.example.com",
  audience: "mobile-app",
  ttl: "15m",
}
```

المفتاح `32` بايتًا على الأقل. غيابه يوقف الإقلاع لأن `accessToken.key` مطلوب حين تُصدر رموزًا. التوقيع HS256 حين يكون المفتاح سرًا مشتركًا. إن كانت خدمات أخرى تتحقق، الصفحة تشرح مفتاح ES256 عامًا. لا تضع السر في Git. خذه من `ConfigService`.

```ts
@Injectable()
export class JwtAuth extends JwtBearerProvider<PublicUser> {
  constructor(
    private readonly users: UsersService,
    registry: AuthenticationRegistry,
  ) {
    super({ realm: "store" });
    registry.registerProvider(this, { order: 1 });
  }

  validate({ sub }: JwtClaims) {
    return sub ? this.users.findById(sub) : null;
  }
}
```

ترتيب `0` للجلسة و `1` للرمز. تكرار الترتيب يفشل الإقلاع. الطلب بلا دليل يرجع `401` مع `WWW-Authenticate: Bearer`. رمز تالف يشرح `error="invalid_token"` كما في RFC 6750.

```ts
return this.tokenService.issue(user.id, {
  method: "password",
  claims: { amr: ["pwd"] },
});
```

`refresh(refreshToken)` يستهلك الرمز. `RefreshTokenError` يصبح `401` من الوحدة. إعادة الاستخدام تلغي العائلة. لا تضع الأدوار المتغيرة داخل `claims` لأن التحديث يعيد نسخها طوال عمر العائلة. `validate` يحمّل المستخدم الحالي من القاعدة.

المخزن `RefreshTokenStore` في الإنتاج، مثل الجلسة. الذاكرة ليست إنتاجًا.

## Mental Model

```text
كلمة مرور -> issue -> access + refresh
Bearer access -> JwtAuth.validate -> User
refresh مرة -> زوج جديد
refresh مسروق مكرر -> إلغاء العائلة
```

## أبسط مثال

```ts
import { Body, Controller, HttpCode, Post, UnauthorizedException } from "@nestjs/common";
import { Public, TokenService } from "@nestjs/authentication";
import { SignInDto } from "./sign-in.dto.js";
import { CredentialsService } from "./credentials.service.js";

@Public()
@Controller("auth/token")
export class TokensController {
  constructor(
    private readonly credentials: CredentialsService,
    private readonly tokens: TokenService,
  ) {}

  @Post()
  @HttpCode(200)
  async issue(@Body() body: SignInDto) {
    const user = await this.credentials.verify(body.email, body.password);
    if (!user) {
      throw new UnauthorizedException("Invalid email or password");
    }
    return this.tokens.issue(user.id, { method: "password", claims: { amr: ["pwd"] } });
  }

  @Post("refresh")
  @HttpCode(200)
  refresh(@Body("refreshToken") refreshToken: string) {
    return this.tokens.refresh(refreshToken);
  }
}
```

## شرح المثال

`@Public()` لأن الإصدار يحدث قبل وجود وصول. `@HttpCode(200)` لأن `@Post` وإلا يرجع `201`، والصفحة تثبّت `200` لهذا الفعل. `verify` هي نفسها فكرة الدخول الموحّد الرسالة. لا تسجّل `refreshToken` في السجلات.

الجلسة تبقى للمتصفح على مساراتها. هذا المتحكم للجوال. الحارس يجرّب الجلسة أولًا ثم الحامل.

## ماذا يحدث داخليًا؟

`issue` يوقّع الوصول بـ `iss` و `aud` و `exp` و `sub`، وينشئ تحديثًا عشوائيًا يخزّن تجزئته. `JwtBearerProvider` يفحص التوقيع والجمهور والمصدر والانتهاء ثم يستدعي `validate`. رجوع `null` يرفض الرمز ولو كان التوقيع صحيحًا، مثلاً إن حُذف المستخدم.

`refresh` يعلّم الرمز مستهلكًا في عملية واحدة في المخزن حتى لا يربح طلبان متزامنان. الفائز يأخذ خليفة. الخاسر أو السارق الذي يعيد الرمز القديم يلغي `familyId`.

## مثال عملي

جوال المكتبة يسجّل الدخول ويحفظ الزوجين في مخزن الجهاز لا في سجل التصحيح. كل `GET /posts` يضع الوصول. قبل انتهاء `15m` يستدعي التحديث. إن استُعيد رمز قديم من نسخة احتياطية للجهاز، العائلة تموت والمستخدم يدخل من جديد. الخروج من كل الأجهزة في الحزمة يستدعي إبطال رموز المستخدم، وهو أوسع من مسح كعكة واحدة.

## أخطاء شائعة

- سر JWT أقصر من الحد أو داخل المستودع.
- تخزين الوصول في query string فيظهر في السجلات.
- وضع `roles` داخل `claims` ثم تغيير دور المستخدم من غير أثر حتى تنتهي العائلة.
- ترتيب المزوّدين نفسه فيفشل الإقلاع.
- معاملة `3600` كساعة في `ttl` هذه الحزمة.

## متى نستخدمه؟

لعميل لا يعتمد على كعكة المتصفح ويقدر أن يخزّن سر التحديث بأمان.

## متى لا نستخدمه؟

لصفحة نفس الموقع التي تكفيها جلسة `HttpOnly`. الرمز في `localStorage` هناك سطح أوسع أمام سكربت الصفحة.

## مقارنة

| كعكة الجلسة | Bearer + refresh |
| --- | --- |
| المتصفح يرسلها | العميل يضع الرأس |
| إبطال بحذف السجل | وصول قصير وتحديث دوار |
| `SessionAuth` ترتيب `0` | `JwtAuth` ترتيب `1` |

## تمارين

### تمرين سهل

أي مسار يصدر الزوجين، وأي مسار يجدّدهما؟

<details>
<summary>الحل</summary>

`POST /auth/token` يصدر، و `POST /auth/token/refresh` يجدّد.

</details>

### تمرين متوسط

لماذا `validate` تجلب المستخدم من القاعدة رغم صحة التوقيع؟

<details>
<summary>الحل</summary>

التوقيع يثبت أن الخادم أصدر الرمز. الجلب يثبت أن الشخص ما زال موجودًا ويقرأ بياناته الحالية لا نسخة قديمة داخل الرمز.

</details>

### تمرين متقدم

لماذا يُرفض تخزين الدور داخل `claims` التي يعيد `refresh` نسخها؟

<details>
<summary>الحل</summary>

العائلة تعيش أطول من الوصول. تغيير الدور في الجدول لا يصل إلى الرموز الجديدة إن كانت تنسخ الدور القديم. `validate` يقرأ الدور من الصف عند كل طلب.

</details>

## أسئلة مراجعة

- ماذا يحدث عند إعادة استخدام تحديث مستهلك؟
- من يفحص `Bearer`، الجلسة أم `JwtAuth`؟
- أين يُحفظ سر التوقيع؟

## الخلاصة

الجوال يحمل وصولًا قصيرًا يجدّده تحديث يُستخدم مرة. المزودان يتعايشان بترتيب ثابت. الدور الحالي يأتي من القاعدة لا من نسخة داخل الرمز.

## ماذا بعد؟

[Passport و @nestjs/passport](./08-passport-overview.md) الوصفة الرسمية التي ما زالت في التوثيق.

## المصادر الرسمية

- NestJS — Authentication: https://docs.nestjs.com/security/authentication
