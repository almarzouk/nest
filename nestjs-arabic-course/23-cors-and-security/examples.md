# أمثلة عملية

ثلاثة حدود على API المقالات. الاستيراد ينتهي بـ `.js` في مشروع ESM.

## مثال 1 — أصل الواجهة

`main.ts` يستدعي `enableCors` بقائمة أصل واحدة من الإعداد، مع `credentials: true` و `methods` التي تشمل `PATCH` و `DELETE`. ثم `useSecurityHeaders()` و `enableCsrfProtection` بنفس الأصل في `trustedOrigins`. أصل CORS لا يُنسخ تلقائيًا إلى CSRF.

## مثال 2 — دخول أبطأ وشكل أضيق

`ThrottlerModule.forRoot` بنافذة `60_000` وحد عام للقراءة، و `@Throttle` على `POST /auth/sign-in` بحد أصغر. `RegisterDto` فيه البريد وكلمة المرور فقط. `forbidNonWhitelisted` يرفض `role`. الخدمة تكتب `role: "reader"` وتستدعي `passwordHasher.hash`.

## مثال 3 — بحث بلا لصق

`SearchService.titles` يستعمل `findMany` مع `contains`. اختبار يمرّر `' OR 1=1 --` ويتوقع نتيجة بحث حرفية. مسار قديم استُبدل كان يبني قالب SQL. عنوان المقال يُعرض في الواجهة كنص لا كـ HTML.

## ماذا تتأكد منه

- `curl` ينجح في `GET` حتى قبل CORS، والمتصفح على أصل آخر يحتاج الرؤوس.
- `POST` من أصل غير موثوق يأخذ `403` قبل Prisma.
- `document.cookie` بلا `__Host-sid`، و `passwordHash` غائب عن JSON.
- لا `$queryRawUnsafe` بإدخال طلب.
