# @Resolver و @Query و @Mutation و @Args

## الهدف من هذا الدرس

ربط حقول الجذر بدوال صنف: المحلل مكان المتحكم في GraphQL، والحجة تصل عبر `@Args`.

## المشكلة التي يحلها

المخطط المولَّد بلا دوال يعرّف الحقول ولا يجلب كتابًا. وضع الجلب داخل `BooksController` ثم استدعاؤه من عميل GraphQL من الشبكة يضاعف الدخول. المحلل يستدعي الخدمة نفسها التي قد يستدعيها REST، ويبقى هو باب العملية.

## الفكرة ببساطة

`@Resolver()` على الصنف. `@Query()` على قراءة و `@Mutation()` على كتابة. `@Args('id')` يقرأ حجة بالاسم. الصنف مزوّد في `providers`، لا في `controllers`. الخدمة تُحقن في الباني كالمعتاد.

## التعريف التقني

```ts
import { Args, ID, Mutation, Query, Resolver } from '@nestjs/graphql';
import { Book } from './book.model.js';
import { BooksService } from './books.service.js';
import { CreateBookInput } from './create-book.input.js';

@Resolver(() => Book)
export class BooksResolver {
  constructor(private readonly books: BooksService) {}

  @Query(() => Book, { nullable: true })
  book(@Args('id', { type: () => ID }) id: string) {
    return this.books.findById(id);
  }

  @Mutation(() => Book)
  createBook(@Args('input') input: CreateBookInput) {
    return this.books.create(input);
  }
}
```

`() => Book` دالة حتى لا تنكسر الأنواع عند الترتيب الدائري. `nullable: true` يطابق مخططًا يعيد `Book` بلا `!`. من غيرها قد يعلن التوليد حقلًا مطلوبًا ثم تعيد الخدمة `null`.

`@Args('input')` اسم الحجة في SDL. يمكن تمرير كائن خيارات للحجة، ومنها `type` عندما لا يكفي انعكاس TypeScript، كما مع `ID`.

## Mental Model

```text
query { book(id: "42") { title } }
  BooksResolver.book("42")
    BooksService.findById
      Book
        حقل title من الكائن إن لم يوجد @ResolveField
```

## أبسط مثال

تسجيل المحلل في الوحدة مع code-first:

```ts
import { Module } from '@nestjs/common';
import { BooksResolver } from './books.resolver.js';
import { BooksService } from './books.service.js';

@Module({
  providers: [BooksResolver, BooksService],
})
export class BooksModule {}
```

`BooksModule` مستوردة من الجذر الذي فيه `autoSchemaFile`. بعد الإقلاع يحوي `schema.gql` الحقلين `book` و `createBook`.

## شرح المثال

`@Resolver(() => Book)` يعلن أن هذا الصنف يتحدث عن نوع `Book`. الاستعلامات على الجذر ما زالت `@Query`. الدالة `book` اسمها في الصنف هو اسم الحقل ما لم تغيّره بالخيار `name`. أبقِ الاسم مطابقًا حتى يسهل إيجاد الدالة من SDL.

الخدمة تعيد كائنًا فيه `id` و `title` على الأقل. المحلل لا ينسق JSON. Apollo يختار الحقول التي طلبها العميل من قيمة الإرجاع.

## ماذا يحدث داخليًا؟

Nest تسجل الدوال المزينة كمحللات حقول الجذر. عند التنفيذ تُبنى الحجج من المتغيرات وتُمرر إلى المعاملات. إن رمى المحلل استثناء HTTP من Nest، الطبقة تحوّله إلى خطأ GraphQL. `null` من `findById` مع `nullable: true` يصبح `data.book: null` بلا استثناء. اختر واحدًا من الأسلوبين وثبّته: الغياب `null` أو الغياب خطأ. لا تخلطهما بين `book` و `author`.

## مثال عملي

قائمة بلا حجج، ثم حجة اختيارية للتصفية في درس الأنواع:

```ts
@Query(() => [Book])
books() {
  return this.books.findAll();
}
```

`[Book]` في الزينة يولّد قائمة. اضبط `nullable` على العناصر إن كان مخططك `[Book!]!` يحتاج ذلك عبر خيارات النوع. إن أعادت الخدمة `null` داخل المصفوفة والمخطط يمنعها، يفشل التنفيذ. أعد مصفوفة فارغة لا `null` عندما لا توجد كتب.

## أخطاء شائعة

- وضع المحلل في `controllers`.
- `@Args('bookId')` بينما العملية ترسل `id`.
- إرجاع كيان Prisma وفيه حقول لا توجد على `@ObjectType`، فيتسرّب شكل القاعدة إن سُلسل الحقل بالخطأ لاحقًا.
- رمي `NotFoundException` تارة و `null` تارة للحقل نفسه.

## متى نستخدمه؟

لكل حقل جذر في كتالوج الكتب: قراءة في `@Query` وكتابة في `@Mutation`، والجسم في الخدمة.

## متى لا نستخدمه؟

لحقل كائن ابن مثل `author` على `Book` عندما يحتاج جلبًا خاصًا. ذلك `@ResolveField` في درس العلاقات، لا `@Query` جديد لكل علاقة.

## مقارنة

| REST | GraphQL في Nest |
| --- | --- |
| `@Controller('books')` | `@Resolver(() => Book)` |
| `@Get(':id')` | `@Query` مع `@Args('id')` |
| `@Post()` و DTO | `@Mutation` و `@Args('input')` |
| `providers` للخدمة فقط | المحلل والخدمة في `providers` |

## تمارين

### تمرين سهل

في أي مصفوفة بالوحدة يُذكر `BooksResolver`؟

<details>
<summary>الحل</summary>

في `providers`، لا في `controllers`.

</details>

### تمرين متوسط

ماذا يطابق `{ nullable: true }` في SDL؟

<details>
<summary>الحل</summary>

حقلًا بلا `!`، فيجوز أن تعيد الدالة `null`.

</details>

### تمرين متقدم

لماذا تُكتب `() => Book` لا `Book` مباشرة في بعض المشاريع؟

<details>
<summary>الحل</summary>

الدالة تؤخر قراءة الصنف حتى يكتمل التحميل، وهذا يلزم عندما تشير الأنواع بعضها إلى بعض.

</details>

## أسئلة مراجعة

- أي زينة للجذر وأي زينة للحجة؟
- من يختار حقول JSON النهائية؟
- أين تعيش قاعدة «الكتاب غير موجود»؟

## الخلاصة

المحلل مزوّد يربط `@Query` و `@Mutation` بالخدمة. `@Args` يسمي الحجة كما في المخطط. الغياب سياسة واحدة للحقل.

## ماذا بعد؟

شكل `Book` و `CreateBookInput` كأصناف مزينة، وهي التي يولّد منها `autoSchemaFile` الأنواع.

## المصادر الرسمية

- NestJS — Resolvers: https://docs.nestjs.com/graphql/resolvers
- NestJS — Quick start: https://docs.nestjs.com/graphql/quick-start
