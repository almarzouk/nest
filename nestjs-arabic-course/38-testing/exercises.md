# تمارين الفصل

## سهل

1. أي مشغّل مع قالب ESM من `nest new`؟
2. ماذا يغلق رحلة Supertest؟
3. أي قاعدة يرفضها حارس التكامل؟

<details>
<summary>الحل</summary>

1. Vitest. CommonJS يبقى على Jest.
2. `app.close()` ويفضَّل في `afterAll`.
3. أي عنوان ليس قاعدة اختبار واضحة مثل `bookstore_test`، وأولها الإنتاج.

</details>

## متوسط

اكتب خطة ثلاث حالات لنشر كتاب: وحدة، تكامل، رحلة. ماذا يكون حقيقيًا في كل واحدة؟

<details>
<summary>الحل</summary>

- وحدة: وهم Prisma ومُصدِر، تتوقع `update` ثم `emitAsync`، وحالة لا إعلان عند الرمي.
- تكامل: `bookstore_test` وصف يُمسح، الخدمة تكتب `PUBLISHED`، الحدث وهم.
- رحلة: Supertest على `POST /books/:id/publish` مع أنبوب وحارس كما في `main.ts`. القص لا يُنفَّذ. الطابور وهم. `init` بلا `listen` إن كفى الخادم الداخلي.

</details>

## متقدم

قالب ESM فشل حقن `BooksService` داخل `TestingModule`، والاختبار استعمل `jest.fn`، و `DATABASE_URL` يشير إلى التطوير، والرحلة تستبدل الحارس ثم تتوقع `403`. صحح الاتجاه.

<details>
<summary>الحل</summary>

- الأوهام `vi.fn` من `vitest` لا تثبيت Jest بجانب القالب.
- فشل الحقن غالبًا إعداد تحويل أسقط `emitDecoratorMetadata`. أرجع إعداد قالب Nest.
- التكامل يتوقف إن لم يكن الاسم `bookstore_test`. لا تمسح قاعدة التطوير.
- رحلة `403` تبقي الحارس الحقيقي وتستبدل Prisma. استبدال الحارس بمن يسمح دائمًا لا يثبت المنع.
- `overrideProvider` قبل `compile`، و `close` بعد الرحلة.

</details>
