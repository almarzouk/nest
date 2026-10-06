# تمارين الفصل

صفحة كتاب حيّة. الحفظ ما زال `POST`.

## سهل

1. أين تُسجَّل البوابة؟
2. أي مكتبة عميل توافق الافتراض؟
3. هل `return` من `join` يبث للغرفة؟

<details>
<summary>الحل</summary>

1. في `providers`.
2. `socket.io-client` إلى فضاء الاسم نفسه.
3. لا. الإرجاع إقرار للمرسل. البث `emit`.

</details>

## متوسط

ارسم تسلسل نشر تعليق يراه قارئ آخر على الكتاب نفسه، من غير حفظ عبر المقبس.

<details>
<summary>الحل</summary>

- القارئ يفتح الصفحة ويتصل بـ `/comments` ثم `emit('join', { bookId })`.
- الكاتب `POST /books/:id/comments`.
- الخدمة تحفظ.
- المتحكم يستدعي `announce`.
- `server.to(bookId).emit('comment', { id, text })`.
- واجهة القارئ تسمع `comment` وتلحق العنصر. إعادة التحميل تعتمد على `GET` لا على المقبس.

</details>

## متقدم

الفضاء صار `/desk` للمسودات. عميل بلا رمز بقي متصلًا لأنه لم يرسل `join`. وعميل مصادق انضم إلى مسودة غيره عبر `bookId`. ماذا تغيّر في الخطاف وفي الرسالة؟

<details>
<summary>الحل</summary>

- `handleConnection` يقرأ `handshake.auth.token` ويستدعي التحقق. الفشل `disconnect` قبل أي استماع.
- النجاح يضع `userId` على `client.data` فقط، لا من جسم الرسالة.
- `join` يستدعي خدمة الإذن `canReadDraft(bookId, userId)` قبل `client.join`.
- الأنبوب يرفض الحقول الزائدة. `WsException` للأخطاء التي يجب أن تراها الواجهة.
- `handleDisconnect` ينظف الربط المحلي إن وُجد.

</details>
