# تمارين الفصل

## تمرين سهل

رتّب: `listen`، تحميل Decorators، `NestFactory.create`.

<details><summary>الحل</summary>

تحميل الملف يشغّل Decorators، ثم `create`، ثم `listen`.

</details>

## تمرين متوسط

`PORT=4000 npm run start:dev`. أي عنوان تفتح؟

<details><summary>الحل</summary>

`http://localhost:4000/` لأن القالب يمرر `process.env.PORT`.

</details>

## تمرين متقدم

اختبار يحتاج التطبيق من غير حجز منفذ على الجهاز. أي خطوة من `bootstrap` يحذف، وما الذي يبقى جاهزًا بعد `create`؟

<details><summary>الحل</summary>

يحذف `listen`. بعد `create` الوحدات والمسارات مسجّلة على خادم المنصة الداخلي ويمكن ضربها من داخل الاختبار.

</details>
