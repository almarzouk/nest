# تمارين الفصل

## تمرين سهل — الأداة

ضعوا الأداة: resilience، idempotency، outbox، workflows، locks.

1. 504 لاقتباس بطيء.
2. إعادة إنشاء مشروع.
3. رسالة مع `COMMIT`.
4. انتظار سبعة أيام.
5. تصدير على نسخة واحدة.

<details>
<summary>الحل</summary>

1 `@nestjs/resilience`. 2 `@nestjs/idempotency`. 3 `@nestjs/outbox` مع `tx`. 4 `@nestjs/workflows`. 5 `@nestjs/locks` بمخزن مشترك.

</details>

## تمرين متوسط — tx

لماذا `outbox.add(message, this.prisma)` يهدم الضمان؟

<details>
<summary>الحل</summary>

العميل الجذر يلتزم وحده وقد تُرسل الرسالة ثم يتراجع نشر المشروع، أو العكس حسب الترتيب. الصفحة تطلب عميل المعاملة التفاعلية وترمي `OutboxTransactionRequiredError` على القاعدة نفسها.

</details>

## تمرين متقدم — ثلاث حاويات

القفل في الذاكرة والتدفق في Postgres. أي جزء ما زال يتكرر وأي جزء ينجو من إعادة التشغيل؟

<details>
<summary>الحل</summary>

التصدير الليلي يتكرر ثلاث مرات لأن كل حاوية ترى قفلها. التدفق المسجّل في القاعدة يُستأنف بلا تكرار الخطوات المكتملة. انقلوا `LockStore` إلى Postgres أو Redis مشترك.

</details>
