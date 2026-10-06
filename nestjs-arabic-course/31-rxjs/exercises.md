# تمارين الفصل

## سهل

1. متى يبدأ عمل `Observable` البارد؟
2. أي معامل يغيّر الجسم وأي معامل يسجّل الزمن؟
3. أي خطأ يصدره `timeout`؟

<details>
<summary>الحل</summary>

1. عند `subscribe`، وNest تشترك إن أرجعت التيار من المعترض.
2. `map` يغيّر، و `tap` يسجّل.
3. `TimeoutError`.

</details>

## متوسط

اكتب سلسلة معترض لمسار ISBN: مهلة 800ms، غلاف `{ data }` عند النجاح، و `504` عند المهلة، مع تمرير `HttpException` كما هو.

<details>
<summary>الحل</summary>

`next.handle().pipe(timeout(800), map((data) => ({ data })), catchError(...))`. داخل `catchError`: إن كان الخطأ `TimeoutError` فأعد `throwError` بـ `GatewayTimeoutException`. إن كان `HttpException` فأعد رميه. لا `of(null)` ولا `retry`.

</details>

## متقدم

مراجع أضاف `retry(2)` و `Subject` على المعترض، واشترك يدويًا ثم أرجع التيار، على `POST /books`. ما الذي يتكرر، وما الذي قد يختلط بين طلبين؟

<details>
<summary>الحل</summary>

- الاشتراك اليدوي مع الإرجاع يشغّل السلسلة مرتين.
- `retry` قد يعيد أثر الإنشاء.
- `Subject` المشترك على مزوّد يدفع أجسام الطلبات إلى مشتركين آخرين.
- المسار الصحيح تيار بارد لكل طلب، بـ `map` أو `tap` فقط، والخدمة `async` بلا إعادة محاولة عمياء.

</details>
