# @ObjectType و @Field و @InputType

## الهدف من هذا الدرس

كتابة صنف الخرج وصنف الدخل اللذين يولّد منهما code-first نوع `Book` ومدخل `CreateBookInput`.

## المشكلة التي يحلها

`@Query(() => Book)` بلا صنف مزين لا يعرف الحقول. إن مررت واجهة TypeScript فقط، التوليد لا يرى ما يُسلسَل. وإن استعملت كيان القاعدة كنوع GraphQL ظهر عمود لا يخص العميل.

## الفكرة ببساطة

`@ObjectType()` على صنف الرد. `@Field()` على كل ورقة تريدها في المخطط. `@InputType()` على صنف الحجج. أسماء الخصائص تصبح أسماء الحقول ما لم تغيّرها. الصنفان ملفان في مجلد الميزة، والمحلل يستقبلهما ويعيدهما.

## التعريف التقني

```ts
import { Field, ID, InputType, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;
}

@InputType()
export class CreateBookInput {
  @Field()
  title: string;
}
```

`@Field()` بلا دالة يكفي لـ `String` و `Boolean` عندما يستطيع الانعكاس قراءة النوع. `ID` ليس نوع TypeScript، لذلك `() => ID`. المصفوفات تُكتب `() => [Book]`.

خاصية بلا `@Field` لا تدخل المخطط في code-first. هذا هو مكان إخفاء `stockInternal` أو تجزئة كلمة المرور.

`@InputType()` لا يُستعمل مكان `@ObjectType()` في قيمة الإرجاع. الدرس 04 شرح الاتجاه في SDL. الزينة هنا هي الترجمة.

## Mental Model

```text
CreateBookInput  --@Args-->  service.create
                                |
Book  <-- @Field id, title -----+
```

حقل على الصنف بلا زينة يبقى في الذاكرة ولا يسافر في GraphQL.

## أبسط مثال

محلل الإنشاء يعيد صنف الخرج لا كائنًا مجهولًا فقط من أجل النوع:

```ts
@Mutation(() => Book)
createBook(@Args('input') input: CreateBookInput) {
  return this.books.create(input);
}
```

الخدمة تعيد `{ id, title }` بشكل `Book`. بعد الإقلاع يظهر في `schema.gql` النوعان وحقل `createBook(input: CreateBookInput!): Book!`.

## شرح المثال

`title` على المدخل مطلوب لأن الخاصية ليست اختيارية ولم تضع `nullable: true` على `@Field`. على الخرج، `id` و `title` مطلوبان. إن أعادت الخدمة `title: undefined` فشل التنفيذ. املأ الحقول المعلنة أو أعلنها قابلة للغياب صراحة.

## ماذا يحدث داخليًا؟

مسح code-first يقرأ الزينة ويبني حقول SDL. أثناء الرد يقرأ Apollo الحقول المطلوبة من الكائن بالاسم. `@Field(() => ID)` لا يحوّل الرقم إلى معرف قاعدة. هو يعلن النوع. التحويل، إن لزم، في الخدمة. المدخل يصل كائنًا عاديًا بعد التحقق من المخطط. إن أضفت `class-validator` لاحقًا فذلك أنبوب على الحجة، وليس أثر `@Field` وحده.

## مثال عملي

حالة الكتاب enum في TypeScript والزينة معًا:

```ts
import { registerEnumType } from '@nestjs/graphql';

export enum BookStatus {
  DRAFT = 'DRAFT',
  PUBLISHED = 'PUBLISHED',
}

registerEnumType(BookStatus, { name: 'BookStatus' });
```

ثم `@Field(() => BookStatus) status: BookStatus` على `Book`. `registerEnumType` لازم لأن enum في TypeScript لا تلتقطه زينة الحقل وحدها كنوع GraphQL باسم مستقر. ضعه في ملف يُستورد عند إقلاع الوحدة.

للتحديث الجزئي `@Field({ nullable: true }) title?: string` على `UpdateBookInput`. الغياب مسموح ويطابق درس المدخلات.

## أخطاء شائعة

- كيان Prisma عليه `@ObjectType()` فيسري كل عمود أضفته للزينة بالخطأ.
- نسيان `registerEnumType` فيصبح النوع مبهمًا أو يفشل التوليد.
- `@InputType()` على الصنف الذي يعيده `createBook`.
- `@Field()` على دالة getter تكشف حقلًا محسوبًا ثقيلًا بلا `@ResolveField` يوضّح التكلفة.

## متى نستخدمه؟

لكل نوع يراه عميل الكتالوج: خرج مستقل عن صف القاعدة، ودخل مستقل عن الخرج.

## متى لا نستخدمه؟

لا تزين كائن الإعداد أو كيان المستخدم الداخلي. الزينة عقد عام. ما لا يملك `@Field` يمكن أن يعيش على صنف آخر بلا `@ObjectType`.

## مقارنة

| الزينة | SDL الناتج | اتجاه البيانات |
| --- | --- | --- |
| `@ObjectType` + `@Field` | `type` | من الخادم |
| `@InputType` + `@Field` | `input` | من العميل |
| `registerEnumType` | `enum` | قيمة مغلقة |

## تمارين

### تمرين سهل

أي زينة تضع `id` في المخطط كـ `ID`؟

<details>
<summary>الحل</summary>

`@Field(() => ID)` على خاصية داخل `@ObjectType()`.

</details>

### تمرين متوسط

خاصية `costPrice` بلا `@Field` على صنف `@ObjectType()`. هل يستطيع العميل طلبها؟

<details>
<summary>الحل</summary>

لا. التوليد لا يراها، فالتحقق يرفض الحقل.

</details>

### تمرين متقدم

لماذا لا تضع `@ObjectType()` على نموذج Prisma مباشرة في هذا المتجر؟

<details>
<summary>الحل</summary>

لأن أعمدة التخزين ستميل إلى أن تصبح حقول واجهة. صنف `Book` يختار ما يُعرض، والخدمة تملأه من الصف.

</details>

## أسئلة مراجعة

- متى تكفي `@Field()` بلا `() => ...`؟
- ماذا يلزم قبل استخدام enum في حقل؟
- كيف تعلن عنوانًا اختياريًا في التحديث؟

## الخلاصة

`@ObjectType` للرد و `@InputType` للدخل و `@Field` لكل ورقة ظاهرة. إخفاء حقل يعني عدم تزيينه. enum يُسجَّل بالاسم.

## ماذا بعد؟

حقل على `Book` لا يأتي من الكائن الأب وحده: `@ResolveField` لعلاقة المؤلف.

## المصادر الرسمية

- NestJS — Resolvers: https://docs.nestjs.com/graphql/resolvers
- NestJS — Quick start: https://docs.nestjs.com/graphql/quick-start
