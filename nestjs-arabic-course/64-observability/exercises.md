# تمارين الفصل

## تمرين سهل — الأداة

لكل سؤال أداة: سجل، مقياس، أثر، health.

1. هل العملية ترد؟
2. أين قضى الطلب وقته داخل Prisma؟
3. هل بطؤ النشر مستمر منذ أمس؟
4. ما `bookId` في الطلب الذي فشل؟

<details>
<summary>الحل</summary>

1 health. 2 أثر. 3 مقياس. 4 سجل مرتبط بـ `requestId`.

</details>

## تمرين متوسط — علم الإنشاء

تريدون مستودع تمرين بلا إرسال إلى Observe، ولاحقًا تجربة اللوحة على فرع. أي علم لكل مرة؟

<details>
<summary>الحل</summary>

`--no-observe` على `nest new` للتمرين. فرع التجربة `--observe` أو إضافة `createObserveModule` و `instrument` يدويًا. لا تحذفوا health في أي الفرعين.

</details>

## تمرين متقدم — تسريب سياق

الخدمة تحفظ `this.requestId = ...` في interceptor. ماذا يحدث لطلبين متزامنين، وما البديل؟

<details>
<summary>الحل</summary>

الطلب الثاني يطمس حقل singleton فيقرأ الأول معرّف الثاني. البديل `AsyncLocalStorage.run` حول بقية السلسلة، مع ترويسة رد لكل طلب.

</details>
