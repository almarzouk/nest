# تمارين الفصل

شباك التذاكر. أجب ثم افتح الحل.

## سهل

1. ماذا يرى العميل عند `throw new Error('boom')` من غير فلتر مخصص؟
2. أي فئة جاهزة لمقعد مبيع، وأي فئة لغياب الصف؟
3. من يُفحص أولًا عند الرمي من المتحكم: فلتر الدالة أم العام؟

<details>
<summary>الحل</summary>

1. `500` ورسالة `Internal server error`.
2. المبيع `ConflictException` أو `SeatTakenException` برمز `409`. الغياب `NotFoundException`.
3. فلتر الدالة، ثم المتحكم، ثم العام.

</details>

## متوسط

صمم الردود لثلاث حالات من غير طمس الرمز:

- `seat` يفشل `@Min(1)`.
- المقعد مبيع والمفتاح `SEAT_TAKEN`.
- لا ترويسة هوية على الحارس.

اذكر الصنف أو الأنبوب والرمز، لا جسم SQL.

<details>
<summary>الحل</summary>

الشكل `400` من `ValidationPipe` عبر `BadRequestException`. المبيع `409` من `SeatTakenException` بجسم فيه `code`. غياب الهوية `401` من `UnauthorizedException` في الحارس. فلتر إن وُجد ينسخ `getStatus()` ولا يكتب `500` لهذه الثلاثة.

</details>

## متقدم

راجع الحوادث:

1. `try/catch` في `refund` يرجع `{ ok: false }` وفلتر المسار لا يعمل.
2. `@Catch()` على الدالة يكتب `500` لاختبار الزينة.
3. `BadRequestException` من Middleware لا يمر على فلتر المتحكم.
4. فلتر عام سليم لم يضف `path` بعد أن رد فلتر الدالة.

ما السبب في كل بند، وما الإصلاح الذي لا يمرر الاستثناء إلى فلتر ثانٍ؟

<details>
<summary>الحل</summary>

1. الخطأ ممسوك. أعد رمي `HttpException` أو لا تلتقطه.
2. القوس الفارغ مسك `BadRequestException`. انسخ `getStatus()` و `getResponse()`.
3. Middleware قبل اختيار المسار. الإصلاح فلتر عام، لا `@UseFilters` على المتحكم.
4. أول إمساك يوقف الباقي. أضف `path` في فلتر الدالة أو اجعله يمتد من العام ويستدعي `super.catch` بعد أن يضبط العام الكتابة.

</details>
