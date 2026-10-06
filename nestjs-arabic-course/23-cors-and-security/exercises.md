# تمارين الفصل

## سهل

1. من يفرض CORS؟
2. أي استدعاء يضع رؤوس Helmet المدمجة؟
3. ما وحدة `ttl` في `@nestjs/throttler`؟

<details>
<summary>الحل</summary>

1. المتصفح عند قراءة سكربت على أصل آخر.
2. `app.useSecurityHeaders()` قبل `listen`.
3. المللي ثانية.

</details>

## متوسط

1. لماذا تفشل `PATCH` عبر الأصل على Fastify رغم `enableCors()` بلا خيارات؟
2. أي طرق يتجاهلها `enableCsrfProtection`؟
3. لماذا `@IsString()` لا يمنع حقن SQL؟

<details>
<summary>الحل</summary>

1. `@fastify/cors` يعلن الطرق الآمنة فقط. أضف `PATCH` في `methods`.
2. `GET` و `HEAD` و `OPTIONS`.
3. القيمة تبقى نصًا قد يُلصق في الاستعلام. الربط بمعامل أو بدالة ORM.

</details>

## متقدم

واجهة على `https://app.example` و API على `https://api.example`. الدخول بالكعكة. مقالات تُعرض عناوينها في React. بحث بعنوان. ملاحظة خاصة تُختم.

1. ماذا يُضبط لأصل الواجهة في CORS و CSRF؟
2. أين تُحفظ كلمة المرور، وأين تُحفظ الملاحظة؟
3. كيف يُبحث عن العنوان؟

<details>
<summary>الحل</summary>

1. `origin` و `credentials` في CORS، ونفس الأصل في `trustedOrigins` لأن القائمة لا تُملأ وحدها. الطرق تشمل `PATCH` و `DELETE`.
2. `PasswordHasher` في `passwordHash`. الملاحظة بـ `aes-256-ctr` مع `iv` جديد ومفتاح خارج المستودع.
3. `contains` أو `$queryRaw` موسوم. لا قالب نصي ولا `sql.raw` بالمدخل.

</details>
