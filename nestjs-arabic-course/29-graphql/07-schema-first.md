# Schema First

## الهدف من هذا الدرس

معرفة الباب الثاني: ملفات SDL هي المصدر، وNest تقرأها من `typePaths` وتستطيع توليد TypeScript منها.

## المشكلة التي يحلها

فريق الواجهة أو شريك خارجي يسلّم `catalog.graphql` كعقد. إعادة كتابته زينة TypeScript ثم توليد ملف آخر تعكس اتجاه الملكية. Schema-first يبقي SDL الذي اتفقتم عليه هو ما ينفّذه الخادم.

هذه الدورة تكتب الميزات code-first. هذا الدرس يمنع خلط البابين، ويعرّفك على الأسماء إن رأيتها في مستودع.

## الفكرة ببساطة

`typePaths` عنوين ملفات `.graphql`. Nest تجمعها في مخطط واحد في الذاكرة. المحللات ما زالت أصنافًا بأسماء الحقول. خيار `definitions` يولّد واجهات أو أصناف TypeScript من تلك الملفات حتى لا تكتب النوع مرتين في الاتجاه المعاكس لـ code-first.

## التعريف التقني

من الصفحة:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
});
```

توليد التعريفات عند الإقلاع:

```ts
definitions: {
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
},
```

الافتراض واجهات. `outputAs: 'class'` يولّد أصنافًا. التوليد عند كل إقلاع قد يزعج التطوير. الصفحة تعرض أيضًا `GraphQLDefinitionsFactory` في سكربت `generate-typings.ts` تشغّله عند الطلب، مع خيار `watch` إن أردت إعادة التوليد عند حفظ `.graphql`.

لا تضبط `autoSchemaFile` و `typePaths` معًا. المصدران سيتعاركان على شكل `Book`.

## Mental Model

```text
src/**/*.graphql     مصدر العقد
  typePaths يجمعها
    resolvers تطابق الأسماء
      definitions تولّد graphql.ts للنوع فقط
```

في code-first السهم معكوس: الصنف يولّد SDL.

## أبسط مثال

ملف بجانب الميزة:

```graphql
type Query {
  book(id: ID!): Book
}

type Book {
  id: ID!
  title: String!
}
```

والجذر:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./src/**/*.graphql'],
});
```

محلل `book` ما زال لازمًا. الملف لا يجلب صفًا من القاعدة. هو يعلن أن الحقل موجود.

## شرح المثال

النمط `./src/**/*.graphql` يجمع مخطط الكتالوج مع مخطط لاحق إن أضفت ملفًا ثانيًا، بشرط ألا يعيد الاثنان تعريف `Query` بشكل يتعارض. التقسيم مفيد عندما يملك فريق ملف المؤلف وآخر ملف الكتاب، ثم يُدمجان في الذاكرة كما تقول الصفحة.

## ماذا يحدث داخليًا؟

عند الإقلاع تُقرأ الملفات وتُبنى AST ثم مخطط تنفيذي. إن وُجد `definitions` تُكتب أنواع TypeScript إلى المسار المحدد. المحلل الذي يعيد شكلًا يخالف SDL يفشل وقت التنفيذ أو وقت التوليد حسب الخلاف. سكربت التوليد المنفصل يناسب CI: العقد يتغير فقط عندما يشغّل أحد `generate`، لا في كل `nest start`.

الصفحة تذكر `ts-node generate-typings` كمثال تشغيل. في مشروع ESM قد يكون التشغيل عبر `tsx` أو عبر ناتج `tsc`. المهم أن السكربت يستدعي `GraphQLDefinitionsFactory` لا أن تحرر `graphql.ts` يدويًا ثم تولّد فوقه.

## مثال عملي

متى ترفض schema-first في متجر هذه الدورة: إن كان الفريق نفسه يكتب المحلل والصنف، فـ `autoSchemaFile` أقصر. متى تقبله: مستودع `schema/` ينشره فريق المنصة، وتطبيق Nest ينفّذ جزء الكتب فقط. عندها ملفات SDL في `typePaths`، والمحللات تملأ الحقول التي التزمت بها، و `definitions` تمنع كتابة `interface Book` يدويًا.

## أخطاء شائعة

- الإبقاء على `autoSchemaFile` «للمعاينة» مع `typePaths`.
- تعديل `src/graphql.ts` المولَّد والاعتقاد أنه مصدر.
- ملف SDL بلا محلل لحقل جذر، فيفشل التنفيذ عند طلبه.
- نمط `typePaths` لا يصل إلى الملفات بعد البناء لأن المسار نسبي لمجلد مختلف.

## متى نستخدمه؟

عندما يكون SDL عقدًا مسلَّمًا من خارج أصناف Nest.

## متى لا نستخدمه؟

بالتوازي مع code-first على الميزة نفسها. اختر مصدرًا واحدًا. مسار الدروس القادمة في هذا الفصل يكتب الزينة على الأصناف.

## مقارنة

| | Code first | Schema first |
| --- | --- | --- |
| تكتب أولًا | `@ObjectType()` | ملف `.graphql` |
| الناتج | `schema.gql` | `graphql.ts` إن فعّلت `definitions` |
| خيار الجذر | `autoSchemaFile` | `typePaths` |
| مسار الدورة | نعم | معرفة بالباب فقط |

## تمارين

### تمرين سهل

أي خيار يدل Nest على ملفات SDL؟

<details>
<summary>الحل</summary>

`typePaths`.

</details>

### تمرين متوسط

ماذا يولّد `definitions.path`؟

<details>
<summary>الحل</summary>

ملف TypeScript من المخطط، واجهات افتراضيًا أو أصنافًا مع `outputAs: 'class'`.

</details>

### تمرين متقدم

لماذا لا تترك التوليد عند كل إقلاع في تطبيق كبير؟

<details>
<summary>الحل</summary>

لأنه يعيد كتابة الملف في كل تشغيل. سكربت `GraphQLDefinitionsFactory` عند تغيير SDL يجعل الناتج متعمدًا وقابلًا للمراجعة.

</details>

## أسئلة مراجعة

- ما اتجاه التوليد مقارنة بـ code-first؟
- هل ملف SDL يستدعي القاعدة؟
- أي خيارين لا يجتمعان؟

## الخلاصة

Schema-first يجعل SDL المصدر و `typePaths` باب القراءة. الدورة تكمل على code-first. لا تشغّل البابين معًا.

## ماذا بعد؟

`@Resolver` و `@Query` و `@Mutation` و `@Args` على مسار code-first.

## المصادر الرسمية

- NestJS — Quick start (Schema first): https://docs.nestjs.com/graphql/quick-start
