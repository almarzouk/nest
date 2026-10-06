# تمارين الفصل

أجب من غير فتح الدروس. ثم افتح الحل.

## سهل

1. ما المفاتيح الأربعة التي ستستخدمها أغلب الوقت داخل `@Module`؟
2. لماذا `AppModule` هي التي تُمرَّر إلى `NestFactory.create`؟

<details>
<summary>الحل</summary>

1. `imports` و `controllers` و `providers` و `exports`.
2. لأن الإقلاع يحتاج وحدة جذرًا يبدأ منها الرسم. الميزات تُربط عبر `imports` انطلاقًا منها.

</details>

## متوسط

ارسم شجرة لوحدات: كتب، طلبات، تدقيق. الطلبات تتحقق من وجود الكتاب وتكتب سطر تدقيق. جدول خصم الكتب يبقى داخل الكتب.

<details>
<summary>الحل</summary>

- `AuditModule`: `providers` و `exports` فيها `AuditService`.
- `BooksModule`: `BooksService` مصدَّر، `DiscountTable` في `providers` فقط، مع المتحكم.
- `OrdersModule`: `imports: [BooksModule, AuditModule]`، و `OrdersService` تحقن الخدمتين المصدَّرتين.
- `AppModule`: تستورد `BooksModule` و `OrdersModule`. استيراد `AuditModule` في الجذر غير لازم إن كانت الطلبات تستوردها.

</details>

## متقدم

ظهرت رسالة أن `BooksService` غير متاحة في سياق `OrdersModule`. اذكر ثلاثة أسباب مختلفة، وكيف تميّز بينها.

<details>
<summary>الحل</summary>

1. `OrdersModule` لا تستورد `BooksModule` ولا جسرًا يعيد تصديرها. العلاج `imports`.
2. `BooksModule` لم تضع الخدمة في `exports`. العلاج إضافة التصدير، لا `@Global()` كأول حركة.
3. الخدمة ليست في `providers` أصلًا، أو سُجّلت في وحدة ثالثة لم تُستورد. العلاج التسجيل في المالك ثم التصدير.

تمييز سريع: افتح ملف الوحدة المالكة وتأكد من `providers` ثم `exports`، ثم ملف المستهلك وتأكد من `imports`. إن كان الصنف مُستوردًا في TypeScript والحاوية ترفض، السبب في بيانات `@Module` لا في مسار الملف.

</details>
