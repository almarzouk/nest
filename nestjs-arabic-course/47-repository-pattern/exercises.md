# تمارين الفصل

## سهل

ما الفرق بين `save(order: Order)` و `Repository<BookEntity>.save`؟

<details>
<summary>الحل</summary>

الأول يحفظ جذر الدومين بعد قواعده. الثاني مساعد TypeORM لصف الكيان كما تعرّفه الأداة، وقد يُستدعى من طبقة بيانات بلا تلك القواعد.

</details>

## متوسط

أين يقف `prisma.book.findMany` لقائمة المتجر؟

<details>
<summary>الحل</summary>

في قارئ أو خدمة كتالوج تعيد DTO. ليس على مستودع جذر الطلب. إن لم يوجد جذر أصلًا، الاستدعاء المباشر في خدمة الوحدة مقبول حتى يظهر ألم العزل.

</details>

## متقدم

صمّم اختبار «لا يُلغى المشحون» واختبار «العمود يُحدَّث» من غير أن يشتركا في نفس البطء.

<details>
<summary>الحل</summary>

الأول `CancelOrder` مع `inMemoryOrders` ويرمي من `Order.cancel`. الثاني `PrismaOrdersRepository` ضد Postgres: `save` ثم `findById` ويتأكد من `status`. لا تعِد سيناريوهات الكيان في اختبار SQL.

</details>
