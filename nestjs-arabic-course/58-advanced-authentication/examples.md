# أمثلة عملية

مسار Google في رف ثم عائلة رموز للجوال. المساران لا يشتركان في سر.

## المثال 1 — زر ثم عودة

الواجهة:

```text
GET /auth/oidc/google/login?redirectTo=/shelf
```

الخادم فيه `google({ clientId, clientSecret })` و `callbackUrl` على `/auth/oidc/:provider/callback`. العودة الناجحة جلسة. `redirectTo` النسبي فقط هو الذي يُتبع.

## المثال 2 — ربط لا إنشاء

قارئ داخل الجلسة يفتح:

```text
GET /auth/oidc/google/login?link=true
```

`resolveUser` يرى `linkTo`. إن كان `sub` لقارئ آخر فـ 409. إن كان البريد المحلي غير مثبت وحاول الدخول من الصفر بلا جلسة، المعالج يعيد `null` ولا يدمج الحسابين.

## المثال 3 — جهاز يضيع

```ts
await tokenService.revoke(lostPhoneRefreshToken);
```

العائلة تتوقف عن التجديد. access الذي صدر قبل دقائق يعيش حتى `ttl` (مثل `15m`) ما لم تضيفوا فحصًا أشد. جلسة المتصفح على حاسوب آخر تبقى إلى أن تُستدعى `sessionService.revoke` لذلك المعرّف مع `userId` الجلسة، أو خروج الكل.

لا تضعوا `AuthGuard('google')` من Passport بجانب `OidcController` في هذا المسار. الوصفة الرسمية تبقى لمن يبني المشروع عليها وحدها.
