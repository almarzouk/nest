# Google

## الهدف من هذا الدرس

توصيل «الدخول بحساب Google» في رف عبر `@nestjs/authentication`: عميل OAuth من نوع تطبيق ويب، ومسار بداية، ومسار عودة، وإعداد `google()`.

## المشكلة التي يحلها

الفريق يعرف OIDC نظريًا ثم يلصق رمز Google الذي ظهر في المتصفح في ترويسة `Authorization` لكل طلبات الكتب. أو يضع `GOOGLE_CLIENT_SECRET` في حزمة الواجهة. أو ينسى عنوان العودة فيطابق Google عنوانًا غير الذي يستمع إليه Nest.

## الفكرة ببساطة

في Google Cloud أنشئوا OAuth client من نوع Web application وسجّلوا عنوان العودة `.../auth/oidc/google/callback`. السر في بيئة الخادم. زر الواجهة رابط عادي إلى `GET /auth/oidc/google/login`. الحزمة تكمل PKCE و `state` و `nonce` وتستبدل الرمز.

## التعريف التقني

الإعداد داخل `AuthenticationModule.forRootAsync`: `oidc.callbackUrl` مثل `` `${APP_URL}/auth/oidc/:provider/callback` `` و `providers.google = google({ clientId, clientSecret })`. `:provider` يخدم أكثر من مزود على نفس المسار. المتحكم ملككم حتى يخضع للبادئة والإصدار والحد:

`OidcService.start(provider, { redirectTo, link })` و `finish(provider, query)`.

`start` يحفظ المعاملة تحت `state` ويضع سرًا في كعكة `HttpOnly` اسمها `__Host-oidc_tx` لعشر دقائق، والـ `state` تجزئة SHA-256 لذلك السر. `finish` يتحقق أن `state` في الاستعلام يطابق التجزئة، ويستبدل الرمز، ويتحقق من ID token، ثم `resolveUser`، ثم `SignInService`. `redirectTo` مسار نسبي فقط حتى لا يصبح التوجيه مفتوحًا. الكعكة والرد يعملان على Express و Fastify عبر محول HTTP من غير `@Req()` حسب الصفحة. Passport يبقى وصفة بديلة على صفحة Recipes، لا طبقة تُضاف فوق `google()` في نفس التدفق.

## Mental Model

```text
زر → GET /auth/oidc/google/login?redirectTo=/shelf
     → حسابات Google
     → GET /auth/oidc/google/callback?code&state
     → جلسة رف ثم توجيه إلى /shelf
```

## أبسط مثال

```ts
import { Controller, Get, Param, Query, Redirect } from '@nestjs/common';
import { OidcService, Public } from '@nestjs/authentication';

@Public()
@Controller('auth/oidc')
export class OidcController {
  constructor(private readonly oidc: OidcService) {}

  @Get(':provider/login')
  @Redirect()
  login(@Param('provider') provider: string, @Query('redirectTo') redirectTo?: string, @Query('link') link?: string) {
    return this.oidc.start(provider, { redirectTo, link: link === 'true' });
  }

  @Get(':provider/callback')
  @Redirect()
  callback(@Param('provider') provider: string, @Query() query: Record<string, string>) {
    return this.oidc.finish(provider, query);
  }
}
```

## شرح المثال

- `@Public()` لأن الحارس العام للحزمة يغلق المسارات، وهذان المساران قبل الجلسة. الربط `link=true` تشترطه الحزمة على جلسة قائمة أصلًا.
- `@Redirect()` يفهم القيمة التي يعيدها `start` و `finish`.
- `provider` سيكون `google` في الزر، والمسار نفسه يقبل مزودًا أضفته بجانب `google()`.

## ماذا يحدث داخليًا؟

1. الزر يفتح `/auth/oidc/google/login`.
2. `start` يتحقق من إعداد `google` ويوجّه إلى Google مع PKCE.
3. Google يعيد المتصفح إلى العنوان المسجّل.
4. `finish` يرفض إن غابت الكعكة أو اختلف `state`، حتى لو نُسخ الرابط من السجل.
5. بعد `resolveUser` تُفتح جلسة رف (أو حالة عامل ثنائي معلّقة) ويُوجَّه المتصفح إلى `redirectTo` أو `/`.

## مثال عملي

عنوان التطوير الذي يُلصق في Google Console يجب أن يطابق `APP_URL`:

```text
http://localhost:3000/auth/oidc/google/callback
```

وفي البيئة:

```text
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
APP_URL=http://localhost:3000
```

`callbackUrl` في الإعداد يستخدم `APP_URL` حتى يختلف الإنتاج عن التطوير من غير تعديل المتحكم. لا تطبعوا `code` في السجل. لا تضعوا السر في مستودع الواجهة.

## أخطاء شائعة

- نوع عميل Google «Desktop» أو عنوان عودة لا يطابق المسار حرفًا.
- تمرير `redirectTo=https://evil.example` وتوقع أن الحزمة تتبعه. الصفحة تقبل المسارات النسبية فقط.
- استخدام access token الراجع من Google كـ `Bearer` على `/books`.
- إضافة `AuthGuard('google')` من Passport في نفس المتحكم «للاحتياط».

## متى نستخدمه؟

عندما يريد قارئ رف الدخول بحساب Google وتفتحون له جلسة على API الكتب.

## متى لا نستخدمه؟

إن كان مسار المشروع Passport عن قصد؛ عندها الاستراتيجية هناك لا هذا المتحكم معًا. ولا لمهمة تستورد كتالوجًا بلا مستخدم جالس.

## مقارنة

| القطعة | مكانها |
| --- | --- |
| client id / secret | بيئة الخادم و Google Console |
| زر الدخول | رابط إلى `login` |
| PKCE و state و nonce | `OidcService` |
| قرار الحساب المحلي | `OidcAccountResolver` |
| Passport | وصفة منفصلة إن كان هو الاختيار الوحيد |

## تمارين

### تمرين سهل

ماذا يفتح زر «الدخول بـ Google» في المتصفح؟

<details>
<summary>الحل</summary>

`GET /auth/oidc/google/login` مع `redirectTo` نسبي إن لزم. ليس عنوان Google مكتوبًا يدويًا في الواجهة، وليس إرسال السر.

</details>

### تمرين متوسط

نُسخ رابط العودة من سجل الوصول وأُعيد فتحه في متصفح آخر. لماذا لا تكتمل الجلسة؟

<details>
<summary>الحل</summary>

لأن `state` يجب أن يطابق تجزئة سر كعكة `__Host-oidc_tx` التي وُضعت عند `start` في المتصفح الذي بدأ. المتصفح الآخر بلا الكعكة، و `finish` يرفض.

</details>

### تمرين متقدم

الإنتاج على `https://api.raf.example` والتطوير على localhost. ماذا يتغير في Google Console وفي `APP_URL` من غير نسخ المتحكم؟

<details>
<summary>الحل</summary>

عنوان عودة لكل بيئة في Console، و `APP_URL` لكل بيئة حتى يُبنى `callbackUrl` الصحيح. المتحكم `:provider/callback` يبقى. السر مختلف لكل عميل OAuth ولا يُشارك بين البيئات في مستودع عام.

</details>

## أسئلة مراجعة

- أي نوع عميل في Google Console؟
- ماذا يفعل `start` قبل أن يغادر المتصفح؟
- لماذا `redirectTo` نسبي؟
- لماذا لا يُرسل سر العميل إلى الزر؟

## الخلاصة

Google في رف عميل ويب مسجّل، و `google()` في إعداد OIDC، ومتحكم يستدعي `start` و `finish`. الكعكة تثبت أن العودة من نفس المتصفح. رمز Google ليس مفتاح `/books`.

## ماذا بعد؟

بعد الدخول يصدر رف رمز تحديث للجوال. الدرس التالي ماذا يحدث عندما يُستبدل هذا الرمز عند كل تجديد.

## المصادر الرسمية

- NestJS — Authentication (Sign in with Google): https://docs.nestjs.com/security/authentication
- NestJS — Passport: https://docs.nestjs.com/recipes/passport
