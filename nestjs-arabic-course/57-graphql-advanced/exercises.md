# تمارين الفصل

## تمرين سهل — من يملك ماذا

ضع الأداة أمام العبارة: DataLoader، complexity plugin، depth rule، persisted safelist، ApolloFederationDriver.

1. رفض سلسلة طولها 12.
2. استعلام مؤلف واحد لقائمة كتب.
3. رفض تجزئة ليست من بناء الواجهة.
4. `resolveReference` لـ `Book`.
5. رفض `limit` كبير قبل Prisma.

<details>
<summary>الحل</summary>

1 قاعدة العمق. 2 DataLoader. 3 القائمة المغلقة. 4 ApolloFederationDriver مع `@ResolveReference`. 5 إضافة التعقيد. لا بند منها يُستبدل بديكور Nest وحيد يشمل البقية.

</details>

## تمرين متوسط — عدّ SQL

`books { author { name } reviews { user { name } } }` على 5 كتب، بلا مجمّع. كم مستوى N+1 ترى؟

<details>
<summary>الحل</summary>

مستوى المؤلف: 5 استعلامات. مستوى المراجعات: 5 إن جُلبت لكل كتاب. ثم مستخدم لكل مراجعة. كل `@ResolveField` يضرب القاعدة على حدة. مجمّع لكل مستوى (مؤلفون، مراجعات بالكتاب، مستخدمون) ينزل كل مستوى إلى استعلام `IN`.

</details>

## تمرين متقدم — إنتاج رف

الواجهة أنهت شاشاتها. ماذا تفتح في الإنتاج وبماذا تبقي GraphiQL؟

<details>
<summary>الحل</summary>

في الإنتاج قائمة تجزئات مغلقة، مع إبقاء التعقيد والعمق على النصوص المسموحة بسبب المتغيرات، و DataLoader على حقول العلاقة. GraphiQL والنص الحر في التطوير فقط. Federation لا يُضاف إلا عندما ينفصل فريق المراجعات فعلًا، ومعه بوابة لا ديكور على الوحدة.

</details>
