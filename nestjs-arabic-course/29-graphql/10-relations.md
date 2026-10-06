# Relations

## الهدف من هذا الدرس

إضافة حقل `author` على `Book` عبر `@ResolveField` عندما لا يكون الاسم محمّلًا على كائن الكتاب.

## المشكلة التي يحلها

وضع `authorName` نصًا ثابتًا على كل صف كتاب ينسخ الاسم. وضع كائن المؤلف كاملًا داخل `BooksService.findById` يجلب العلاقة حتى عندما تطلب البطاقة `title` فقط. الحقل العلائقي يُحسب إذا طلبته العملية.

## الفكرة ببساطة

`Book` يحمل `authorId` بلا `@Field`، أو يحمل معرفًا داخليًا. المخطط يعلن `author: Author!`. دالة `@ResolveField('author')` تستقبل الكتاب الأب وتستدعي `AuthorsService`. إن لم يطلب العميل `author`، الدالة لا تُستدعى.

## التعريف التقني

```ts
@ObjectType()
export class Author {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;
}

@ObjectType()
export class Book {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;

  @Field(() => Author)
  author: Author;
}
```

المحلل:

```ts
@ResolveField(() => Author)
author(@Parent() book: Book & { authorId: string }) {
  return this.authors.findById(book.authorId);
}
```

`@Parent()` قيمة الحقل الأب كما أعادها محلل `book` أو `books`. `@ResolveField` على `BooksResolver` المزين بـ `@Resolver(() => Book)` تربط الدالة بنوع `Book`.

استدعاء الخدمة مرة لكل كتاب في قائمة هو شكل N+1. الدرس 12 يلمّح إلى DataLoader. هنا الهدف أن ترى مكان الجلب، لا أن تحلّ التكلفة بعد.

## Mental Model

```text
books() -> Book[]  فيها authorId
  لكل كتاب، إن وُجد author في العملية:
    ResolveField.author(parent)
```

بلا `author` في النص، الحلقة لا تعمل.

## أبسط مثال

عملية تفاصيل:

```graphql
query {
  book(id: "42") {
    title
    author { name }
  }
}
```

استعلام البطاقة `{ book(id: "42") { title } }` لا يستدعي `AuthorsService`.

## شرح المثال

الخدمة التي تعيد الكتب تُرجع `{ id, title, authorId }` والكائن يكفي `@Field` الظاهر لأن `author` سيُستبدل بناتج `@ResolveField`. لا تملأ `author: null` وتمنع الحقل. اترك الدالة هي المصدر. `authorId` بلا `@Field` لا يظهر للعميل، ويبقى متاحًا للدالة لأن الكائن في الذاكرة أغنى من المخطط.

## ماذا يحدث داخليًا؟

Apollo يمشي الشجرة. بعد حل `book` يحل الحقول المطلوبة. `title` قراءة خاصية. `author` استدعاء الدالة المسجّلة مع الأب. الناتج `Author` يُحل `name` منه. إن أعادت الدالة `null` والحقل `Author!` فشل التنفيذ. إن كان الكتاب يتيمًا، إمّا تمنع ذلك في البيانات وإما تعلن `Author` قابلًا للغياب. قائمة `books` تشغّل الدالة لكل عنصر طُلب له المؤلف. هذا سلوك صحيح وظيفيًا وخطير على القاعدة إن كانت الدالة استعلامًا لكل نداء.

## مثال عملي

اتجاه عكسي على `AuthorsResolver`: `@ResolveField(() => [Book])` يستدعي `books.findByAuthorId(author.id)`. اتجاه واحد يكفي للمنتج الأول. `() => Author` تؤجل القراءة إن دار الاستيراد بين الصنفين. التكلفة تتبع نص العملية.

## أخطاء شائعة

- `include: { author: true }` في كل `findById` رغم أن الحقل محلل مستقل، فتدفع التكلفة مرتين.
- كشف `authorId` في المخطط مع `author` بلا سبب للعميل.
- `@ResolveField` على محلل غير مربوط بـ `@Resolver(() => Book)`.
- اعتبار العلاقة «محلولة» لأن البطاقة سريعة. القائمة هي الاختبار.

## متى نستخدمه؟

عندما يملك الحقل مصدرًا غير كائن الأب: مؤلف، عدد مراجعات، توفر يُحسب.

## متى لا نستخدمه؟

لحقل منسوخ أصلًا على الأب مثل `title`. `@Field` يكفي. ولإصلاح N+1، لا تضع `JOIN` عشوائيًا في كل محلل أب قبل أن تقيس. التلميح في درس DataLoader.

## مقارنة

| الحقل | من يملأه |
| --- | --- |
| `title` | كائن الكتاب العائد من `@Query` |
| `author` | `@ResolveField` عند الطلب |
| `authorId` | ذاكرة الخدمة، بلا `@Field` |

## تمارين

### تمرين سهل

أي زينة تستلم كائن الكتاب داخل محلل المؤلف؟

<details>
<summary>الحل</summary>

`@Parent()`.

</details>

### تمرين متوسط

لماذا لا تُستدعى `author()` في استعلام البطاقة؟

<details>
<summary>الحل</summary>

لأن العملية لم تطلب الحقل. Apollo لا يشغّل محلل حقل غائب عن الشجرة.

</details>

### تمرين متقدم

قائمة 20 كتابًا مع `author { name }` وكل نداء يفتح استعلامًا. ماذا سمّيت المشكلة، وأين تُعالج بعمق؟

<details>
<summary>الحل</summary>

N+1. هذا الفصل يلمّح إلى DataLoader لكل طلب. العمق في `57-graphql-advanced`.

</details>

## أسئلة مراجعة

- أين يُعلن الحقل وأين تُكتب دالته؟
- لماذا `authorId` بلا `@Field`؟
- ماذا يحدث لقائمة إن كان المحلل استعلامًا لكل أب؟

## الخلاصة

العلاقة `@ResolveField` تعمل فقط إن طُلب الحقل. الأب يحمل مفتاحًا داخليًا. تكلفة القائمة ليست منتهية بهذا الدرس.

## ماذا بعد؟

من يرى الطلب داخل المحلل: context، والحارس على باب GraphQL.

## المصادر الرسمية

- NestJS — Resolvers: https://docs.nestjs.com/graphql/resolvers
- NestJS — Quick start: https://docs.nestjs.com/graphql/quick-start
