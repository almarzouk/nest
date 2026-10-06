# تمارين الفصل

أجب على كتالوج كتب. المسار code-first مع Apollo على Express.

## سهل

1. أي جذر للقراءة وأي جذر لإنشاء كتاب؟
2. أي خيار يولّد `schema.gql`؟
3. أي مفتاح اشتراك تكتبه في الإعداد الحالي؟

<details>
<summary>الحل</summary>

1. `Query` و `Mutation`.
2. `autoSchemaFile` بمسار الملف، مع `driver: ApolloDriver`.
3. `'graphql-ws'`.

</details>

## متوسط

صمم نوع `Book` ومدخل إنشاء واستعلام بطاقة لا يجلب المؤلف. اذكر الزينة لا SQL.

<details>
<summary>الحل</summary>

- `@ObjectType() class Book` بحقول `id` كـ `ID` و `title` كـ `String`. `authorId` بلا `@Field`.
- `@InputType() class CreateBookInput` فيه `title` فقط.
- `@Query(() => Book, { nullable: true }) book(@Args('id', { type: () => ID }) id: string)`.
- العملية: `query { book(id: "42") { title } }`.
- `@ResolveField` للمؤلف موجود لكن هذه العملية لا تشغّله.

</details>

## متقدم

قائمة عشرين كتابًا مع المؤلف أظهرت 21 استعلامًا. الحارس على `createBook` يقرأ `switchToHttp()`. والاشتراك من عميل قديم لا يتصل. أصلح الاتجاه لا الشيفرة الكاملة.

<details>
<summary>الحل</summary>

- N+1 من `@ResolveField` لكل عنصر. DataLoader جديد داخل `context` لكل عملية، ودالة دفعة تعيد النتائج بترتيب المفاتيح. التفصيل في فصل 57. لا محمّل singleton.
- الحارس يستخدم `GqlExecutionContext.create(context).getContext()` لأن `context` في `forRoot` هو الذي يمرر `req`.
- العميل ينتقل إلى بروتوكول `graphql-ws`. مفتاح `subscriptions-transport-ws` لم يعد إعدادًا صالحًا.
- الهوية تُستنتج من الطلب بعد التحقق، ولا تُقبل حقلًا في `CreateBookInput`.

</details>
