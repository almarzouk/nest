# @nestjs/authentication

## الهدف من هذا الدرس

تسجيل حزمة المصادقة الحالية: حارس عام، مزوّد جلسة، وتسجيل دخول يضع الكعكة من غير `@Req()`.

## المشكلة التي يحلها

حارس منزلي على كل متحكم يُنسى على مسار جديد. قراءة الكعكة داخل المتحكم تربط الخدمة بإطار Express أو Fastify. تخزين الجلسات في الذاكرة ينهار عند نسختين من التطبيق، والحزمة ترفض ذلك أصلًا حين `NODE_ENV=production`.

## الفكرة ببساطة

`AuthenticationModule.forRoot` خيارات بيانات: أعمار الجلسة والأسرار لاحقًا. مزوّد تكتبه، مثل `SessionAuth`، يسجّل نفسه في `AuthenticationRegistry` ويعرف كيف يحوّل الدليل إلى مستخدم. الحارس العام يمر على المزوّدات. المسار المفتوح يُعلَّم `@Public()`. `SignInService.signIn` ينشئ الجلسة ويكتب الكعكة.

## التعريف التقني

من صفحة Authentication. المستخدم الذي تراه بقية التطبيق بلا تجزئة:

```ts
export interface User {
  id: string;
  email: string;
  emailVerified: boolean;
  roles: Array<"customer" | "staff" | "admin">;
}
```

مثال الدورة كان `id` عددًا. الحزمة تعرض `id` نصيًا. طابق نوع مشروعك. الفكرة أن `passwordHash` لا يغادر المستودع.

`SessionAuth` يمتد `SessionCookieProvider<User>` ويسجّل `registry.registerProvider(this)` ويستدعي `validate(session)` التي تجلب المستخدم بـ `session.userId`. رجوع `null` ينهي الجلسة إن حُذف المستخدم.

```ts
AuthenticationModule.forRoot({
  session: {
    absoluteTtl: "14d",
    idleTtl: "3d",
  },
});
```

الأعمار مللي ثانية أو نصوص مثل `14d`، بحد أعلى مئة سنة. رقم صغير مثل `3600` هنا ثوانٍ قليلة لا ساعة، بعكس `@nestjs/jwt`.

الحارس بعد الإقلاع يرفض المجهول بـ `401`. `@Public()` على متحكم التسجيل والدخول. `PasswordHasher` و `SignInService` مزوّدان جاهزان. `signIn(user.id, { method: "password" })` يضع كعكة `__Host-sid`، ويحذف جلسة الكعكة السابقة، ويرفض تزييف الدخول من موقع آخر.

في الإنتاج يلزم مخزن `SessionStore` تكتبه أنت، و Drizzle أو TypeORM في الصفحة أمثلة للمخزن لا مسار دورة Prisma. طبّق الواجهة على `PrismaService` إن أكملت الإنتاج. حتى ذلك، الذاكرة للتطوير فقط، والحزمة ترفض الإقلاع بها في الإنتاج.

`forRootAsync` حين تُقرأ الأسرار من `ConfigService`.

ترتيب هذا الفصل: الحزمة أولًا. Passport لاحقًا كوصفة ما زالت قائمة.

## Mental Model

```text
forRoot(options)
SessionAuth.validate -> User | null
حارس عام
  @Public -> مرور
  بلا مستخدم -> 401
SignInService.signIn -> كعكة
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { AuthenticationRegistry, SessionCookieProvider, type SessionRecord } from "@nestjs/authentication";
import { UsersService } from "../users/users.service.js";
import type { PublicUser } from "../users/public-user.js";

@Injectable()
export class SessionAuth extends SessionCookieProvider<PublicUser> {
  constructor(
    private readonly users: UsersService,
    registry: AuthenticationRegistry,
  ) {
    super();
    registry.registerProvider(this);
  }

  validate(session: SessionRecord) {
    return this.users.findById(session.userId);
  }
}
```

## شرح المثال

`findById` ترجع الشخص العام أو `null`، بلا تجزئة. `super()` بلا وسائط لأن المزوّد الأساسي يأخذ `SessionService` بحقن خاصية، لذلك البنّاء يذكر اعتماداتك أنت. التسجيل في `AuthenticationRegistry` هو ما يضع المزوّد في السلسلة. نسيان `registerProvider` يعني حارسًا بلا طريقة إثبات.

ضع الصنف في `providers` لوحدة `AuthModule` التي تستورد `UsersModule`.

## ماذا يحدث داخليًا؟

عند الإقلاع، بعد البنّاءات، السجل يُقفل ويسجّل أسماء المزوّدات. لكل طلب يشغّل الحارس السلسلة. مزوّد الجلسة يقرأ الكعكة، يحمّل السجل، ويستدعي `validate`. أول مزوّد يرجع مستخدمًا يفوز. بلا كعكة صالحة فالنتيجة `401` على المسار غير العام.

الكعكة `HttpOnly` و `Secure` و `SameSite=Lax`. على `localhost` تقبل المتصفحات الحديثة `Secure`. على HTTP خارج ذلك اضبط `cookie.secure` كما تشرح الصفحة، فيصبح الاسم `sid`.

## مثال عملي

`POST /auth/sign-up` و `POST /auth/sign-in` عليهما `@Public()`. بعد الدخول، `GET /posts` من غير كعكة `401`. مع الكعكة يصل إلى `PostsService`. `POST /posts` يأخذ المؤلف من المستخدم الذي ثبّته المزوّد، لا من الجسم. أبقِ قائمة المنشورات العامة `@Public()` أو `@Authenticate({ optional: true })` إن كان الكتالوج مفتوحًا، وهذا يُضبط حين تلمس التفويض. الآن يكفي أن تميّز المسار المفتوح من المغلق.

## أخطاء شائعة

- حماية مسار التسجيل بالحارس العام من غير `@Public()`.
- اعتبار `3600` ساعة في خيارات هذه الحزمة.
- نشر الإنتاج بمخزن الذاكرة. الإقلاع يفشل عمدًا.
- قراءة `req.cookies` في الخدمة بدل `validate`.

## متى نستخدمه؟

لمسار الدورة الحالي على Nest 12 حين تريد جلسات وتجزئة ورموزًا من قسم Security.

## متى لا نستخدمه؟

لنسخ وصفة Passport قائمة تعمل وتريد فقط فهمها. ذلك الدروس التالية، لا حذفها من المشروع القديم في اليوم نفسه.

## مقارنة

| حارس على متحكم واحد | الحزمة |
| --- | --- |
| يُنسى على مسار جديد | عام مع `@Public()` |
| الكعكة في المتحكم | `SignInService` يكتبها |
| بلا مخزن موحّد | واجهات مخازن تكتبها أنت |

## تمارين

### تمرين سهل

ماذا يرجع `validate` إن حُذف المستخدم وما زالت الكعكة حية؟

<details>
<summary>الحل</summary>

`null`. الصفحة تقول إن ذلك ينهي الجلسة، ولا يبقى الطلب موثّقًا.

</details>

### تمرين متوسط

لماذا `forRoot` لا يستقبل صنف المزوّد؟

<details>
<summary>الحل</summary>

الخيارات بيانات. المزوّد مزوّد في وحدتك يسجّل نفسه حتى يستطيع حقن `UsersService`.

</details>

### تمرين متقدم

`NODE_ENV=production` والإقلاع يرفض الجلسات. ماذا ينقص؟

<details>
<summary>الحل</summary>

مخزن `SessionStore` في القاعدة عبر الواجهة التي توثّقها الحزمة، لا الذاكرة الافتراضية.

</details>

## أسئلة مراجعة

- من يسجّل مزوّد الجلسة في السلسلة؟
- أي مسارات تُعلَّم `@Public()`؟
- ما اسم خدمة إصدار الجلسة؟

## الخلاصة

`@nestjs/authentication` حارس عام ومزوّد تكتبه وتسجيل دخول يضع الكعكة. الإنتاج يحتاج مخزنًا. المستخدم الحالي هو ما تقرأه المسارات بعد ذلك.

## ماذا بعد؟

[قراءة المستخدم الحالي](./06-current-user.md) من المعامل أو من السياق.

## المصادر الرسمية

- NestJS — Authentication: https://docs.nestjs.com/security/authentication
