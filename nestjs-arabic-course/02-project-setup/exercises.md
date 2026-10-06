# تمارين الفصل

## تمرين سهل

ما حدود Node لتشغيل Nest 12 على الخط 20، وما أمر معرفة نسختك؟

<details><summary>الحل</summary>

20.19 أو أحدث. الأمر `node -v`.

</details>

## تمرين متوسط

ولّدت خدمة `nest g service health` ولم تُستدعَ. ما أول مكان تفحصه في الوحدة؟

<details><summary>الحل</summary>

مصفوفة `providers` في الوحدة التي ولّد داخلها CLI. إن أنشأت الملف يدويًا فقد يكون غائبًا عن المصفوفة.

</details>

## تمرين متقدم

مشروع ESM يفشل على `Cannot find module './app.module'` بعد حذف الامتداد. ما التعديل، ولماذا القالب يكتب `.js`؟

<details><summary>الحل</summary>

أرجع `./app.module.js`. ESM يحمّل مسار الناتج، و TypeScript يربط الامتداد `.js` بالمصدر `.ts`.

</details>
