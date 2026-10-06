# تمارين الفصل

## سهل

1. أي رأس يعيد المتصفح للكعكة؟
2. أي صفة تخفي `__Host-sid` عن JavaScript؟
3. هل `userId=4` جلسة آمنة؟

<details>
<summary>الحل</summary>

1. `Cookie`.
2. `HttpOnly`.
3. لا. الرقم يُزوَّر. الجلسة مفتاح عشوائي يشير إلى سجل.

</details>

## متوسط

1. لماذا `maxAge` في Nest ليس كما في `res.cookie`؟
2. ماذا يخزّن الخادم بدل رمز الكعكة الخام في الحزمة؟
3. لماذا لا يذهب JWT واجهة الموقع إلى `localStorage`؟

<details>
<summary>الحل</summary>

1. Nest يعدّ الثواني و Express يعدّ المللي ثانية في تلك الدالة.
2. تجزئة SHA-256 للرمز.
3. السكربت يقرأه عند XSS. كعكة الجلسة `HttpOnly` لا تُقرأ هكذا.

</details>

## متقدم

إنتاج فيه نسختان من API، و `express-session` بمخزنه الافتراضي، و `AuthenticationModule` بلا `SessionStore`، والواجهة تحفظ `accessToken` في `localStorage` مع كعكة `__Host-sid`.

1. ماذا تفشل النسختان؟
2. أي دليل تبقيه للواجهة؟
3. أي دليل للجوال من غير لمس `localStorage` في الصفحة؟

<details>
<summary>الحل</summary>

1. الجلسات في ذاكرة كل عملية فلا تُشارَك. الحزمة أصلًا ترفض ذاكرة الجلسة في الإنتاج. الوسيط الافتراضي غير مناسب للإنتاج حسب صفحته.
2. كعكة `__Host-sid` مع مخزن مشترك في القاعدة، وصفات `HttpOnly` و `Secure` و `Lax`.
3. `Authorization: Bearer` مع تحديث دوار في مخزن الجهاز، لا في JavaScript الصفحة. احذف الرمز من `localStorage` على الواجهة.

</details>
