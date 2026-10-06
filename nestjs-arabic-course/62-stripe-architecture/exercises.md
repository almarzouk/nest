# تمارين الفصل

## تمرين سهل — من يكتب الحالة

رتّبوا: إنشاء الجلسة، فتح success، `constructEvent` ناجح بحالة `active`.

<details>
<summary>الحل</summary>

الكتابة المحلية إلى `active` في الخطوة الثالثة فقط وبعد ربط العميل بالمنظمة. الخطوتان الأوليان لا تغيّران الحالة إلى مدفوع.

</details>

## تمرين متوسط — توقيع

اختبار يمرر `JSON.stringify(body)` إلى `constructEvent` ويفشل مع توقيع Stripe الصحيح. ما المدخل البديل؟

<details>
<summary>الحل</summary>

`req.rawBody` بعد `{ rawBody: true }`، مع ترويسة `Stripe-Signature` وسر الوجهة.

</details>

## تمرين متقدم — ضغطة مزدوجة وحدث مكرر

صمموا مفتاحين مختلفين لسيناريو: زر الترقية أُعيد خلال ثانية، ثم `evt_9` وصل مرتين.

<details>
<summary>الحل</summary>

`idempotencyKey` الثابت `checkout:org:plan` على `sessions.create` حتى ترجع نفس الجلسة. قيد فريد على `evt_9` قبل `syncFromStripe` والبريد حتى لا يتكرر الأثر. success URL لا يحمل أي مفتاح كتابة.

</details>
