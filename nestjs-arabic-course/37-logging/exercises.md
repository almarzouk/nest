# تمارين الفصل

## سهل

1. أي مستوى لطابور طويل ما زال يعمل؟
2. ماذا يفعل `bufferLogs`؟
3. أي حقول تصف الطلب الوارد؟

<details>
<summary>الحل</summary>

1. `warn`.
2. يؤجل أسطر الإقلاع حتى `useLogger`.
3. `method` و `path` و `statusCode` و `durationMs`، ومعها `requestId` بعد درس المعرف.

</details>

## متوسط

صمم سطر فشل ISBN وسطر `POST /books` الناجح بحيث يجدهما المجمع معًا.

<details>
<summary>الحل</summary>

الوسيط يضع `requestId` في `AsyncLocalStorage`. سطر HTTP من المعترض بلا جسم وبلا استعلام. سطر `IsbnService` فيه `isbn` و `code` و `requestId` نفسه بلا ترويسات. البحث على `requestId` يعيد الاثنين.

</details>

## متقدم

القص فشل بعد دقيقتين من الرد. ما الذي يجب أن يكون قد نُسخ عند `add`، ولماذا لا يكفي التخزين المحلي داخل العامل؟

<details>
<summary>الحل</summary>

`requestId` داخل حمولة `{ bookId, coverKey, requestId }`. العامل سياق لاحق أو عملية أخرى فلا يرى `AsyncLocalStorage` الخاص بالطلب. `jobId` يبقى `resize:${bookId}` ولا يُستبدل بالمعرف. السجل في `process` يطبع الحقلين. لا كلمة مرور ولا مفتاح ناشر في أي سطر.

</details>
