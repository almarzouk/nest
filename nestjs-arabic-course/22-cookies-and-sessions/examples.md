# أمثلة عملية

المتصفح على المكتبة. الجوال ليس في هذه الأمثلة.

## مثال 1 — سمة بلا هوية

`PUT /preferences/theme` يستدعي `setCookie` باسم `library_theme` و `sameSite: "lax"` و `maxAge` ثلاثين يومًا بالثواني. `GET` يقرأ `@Cookies("library_theme")` ويعيد `light` عند الغياب. الكعكة ليست `HttpOnly` إن كان السكربت يبدل السمة قبل الرد. لا بريد فيها.

## مثال 2 — دخول

`POST /auth/sign-in` ناجح يضع `__Host-sid` عبر `SignInService`: `HttpOnly` و `Secure` و `SameSite=Lax`. الطلب التالي إلى `GET /me` يرسل الكعكة وحدها. الخروج يحذف السجل ويمسح الكعكة. القيمة ليست `userId`.

## مثال 3 — ما لا يُجمع

لا `express-session` لحفظ `user` بجانب الحزمة. عداد `GET /stats/visits` فقط إن بقي الوسيط، مع سر من البيئة و `resave: false` و `saveUninitialized: false`، وبمعرفة أن ذاكرة العملية للتجربة.

## ماذا تتأكد منه

- أداتا تطوير تعرضان كعكة الجلسة `Secure` على `localhost`.
- `document.cookie` لا يظهر `__Host-sid`.
- `maxAge` في `setCookie` بالثواني.
- إنتاج الجلسات ليس مخزن الذاكرة.
