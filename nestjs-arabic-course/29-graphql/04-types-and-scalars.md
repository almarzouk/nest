# Object Types و Input Types و Scalars و Enums

## الهدف من هذا الدرس

تمييز أربعة أشكال في المخطط: كائن يُعاد، إدخال يُستقبل، ورقة scalar، وقيمة مغلقة enum.

## المشكلة التي يحلها

استخدام `Book` نفسه جسمًا للرد وحجة للإنشاء يسمح للعميل بإرسال `id` و `createdAt` في الكتابة. المخطط عندها يوثّق حقولًا لا يقبلها المجال. والقائمة المفتوحة من النصوص لحالة الكتاب تسمح بـ `"shipped"` في حقل يتوقع `DRAFT` أو `PUBLISHED`.

## الفكرة ببساطة

`type` كائن خرج. `input` كائن دخل، ولا يُستخدم مكان النوع في الرد. الـ scalar ورقة لا حقول لها. الـ enum مجموعة أسماء يتحقق منها المحرك قبل دالتك.

## التعريف التقني

```graphql
enum BookStatus {
  DRAFT
  PUBLISHED
}

type Book {
  id: ID!
  title: String!
  status: BookStatus!
}

input CreateBookInput {
  title: String!
  status: BookStatus = DRAFT
}

type Mutation {
  createBook(input: CreateBookInput!): Book!
}
```

الـ scalars المدمجة: `Int` و `Float` و `String` و `Boolean` و `ID`. `ID` تُسلسَل كنص لكنها تعني معرفًا. لا يوجد `Date` في المواصفة الأساسية. التاريخ إما `String` بعقد ISO، وإما scalar مخصص تكتبه لاحقًا. في هذا الفصل أبقِ التواريخ نصًا حتى لا يتشتت التسجيل.

`input` لا يحتوي حقولًا من نوع كائن خرج. يمكن أن يحتوي scalars و enums ومدخلات أخرى. القيمة الافتراضية `= DRAFT` تُستخدم إن غاب الحقل في المتغيرات.

## Mental Model

```text
عميل -- CreateBookInput --> Mutation.createBook
                              |
                              Book  --> عميل
```

السهم الداخل مدخل. السهم الخارج كائن. لا تعكسهما.

## أبسط مثال

عملية الإنشاء:

```graphql
mutation Create($input: CreateBookInput!) {
  createBook(input: $input) {
    id
    status
  }
}
```

```json
{ "input": { "title": "NestJS in Practice" } }
```

`status` غاب فاستخدم المخطط `DRAFT`. الرد يطلب `id` الذي ولّدته الخدمة، وهو غير موجود في المدخل.

## شرح المثال

العميل لا يرسل `id`. الخدمة تولّده. `PUBLISHED` قيمة يجوز إرسالها لأنها في الـ enum. `"draft"` بأحرف صغيرة تفشل التحقق لأن أسماء enum في SDL حساسة للحالة كما كُتبت. أبقِ الأسماء كما في المجال حتى لا تترجم في كل محلل.

## ماذا يحدث داخليًا؟

أثناء التحقق، المحرك يطابق كائن المتغيرات بشكل `CreateBookInput`. حقل زائد مثل `id` يفشل الطلب في النمط المعتاد للمحركات (رفض الحقول المجهولة على الإدخال). ثم تُستدعى دالة `createBook` بالكائن المتحقق. الإرجاع يجب أن يشبه `Book`: إن نقص `status` فشل التنفيذ لا التحقق. لذلك DTO الدخول في Nest لاحقًا يطابق `input`، وصنف الخرج يطابق `type`.

## مثال عملي

تحديث جزئي مدخل آخر، لا إعادة استخدام الإنشاء:

```graphql
input UpdateBookInput {
  title: String
  status: BookStatus
}
```

الحقول بلا `!` لأن الغياب يعني «لا تغيّر». `CreateBookInput` ما زال يفرض `title`. دمج الاثنين في مدخل واحد يجعل العنوان اختياريًا عند الإنشاء أو إجباريًا عند التعديل.

قائمة الرف `books(status: BookStatus): [Book!]!` تأخذ enum مباشرة. إن تجاوزت الحجج اثنتين فاجمعها في `input`.

## أخطاء شائعة

- `type CreateBookInput` بدل `input`.
- إرجاع `CreateBookInput` من المحلل.
- تمثيل الحالة بـ `String` ثم مقارنتها في كل خدمة.
- اختراع scalar للتاريخ في أول مخطط قبل أن يستقر شكل ISO في الفريق.

## متى نستخدمه؟

`input` لكل كتابة لها أكثر من حجة أو ستنمو. `enum` لكل مجموعة مغلقة. `type` لما يقرأه العميل.

## متى لا نستخدمه؟

لا تضع `passwordHash` على `type User` لأنه موجود في الجدول. المخطط واجهة، لا انعكاسًا لصف Prisma.

## مقارنة

| الكلمة | الاتجاه | في المتجر |
| --- | --- | --- |
| `type Book` | خرج | ما يُرسم |
| `input CreateBookInput` | دخل | ما يُقبل عند الإنشاء |
| `enum BookStatus` | قيمة مغلقة | `DRAFT` أو `PUBLISHED` |
| `String` / `ID` | ورقة | عنوان ومعرف |

## تمارين

### تمرين سهل

هل يُعاد `CreateBookInput` في حقل الرد؟

<details>
<summary>الحل</summary>
لا. الرد `Book` أو نوع خرج آخر. المدخل للعملية فقط.
</details>

### تمرين متوسط

لماذا `UpdateBookInput.title` بلا علامة `!`؟

<details>
<summary>الحل</summary>
لأن التحديث الجزئي يفسّر الغياب كإبقاء القيمة. الإنشاء هو الذي يفرض العنوان.
</details>

### تمرين متقدم

العميل أرسل `status: "draft"`. أين يُرفض الطلب؟

<details>
<summary>الحل</summary>
عند التحقق من المخطط. `BookStatus` لا يحتوي هذا الرمز. المحلل لا يُستدعى.
</details>

## أسئلة مراجعة

- ما الـ scalars الخمسة الأساسية؟
- لماذا لا تضع `id` في مدخل الإنشاء؟
- متى تجمع الحجج في `input`؟

## الخلاصة

الخرج `type` والدخل `input` والحالة المغلقة `enum` والورقة scalar. المخطط يمنع حقول الجدول التي لا تريد كشفها.

## ماذا بعد؟

تركيب هذا المخطط داخل Nest مع Apollo، وما زالت الزينة في الدرس الذي يلي الإعداد.

## المصادر الرسمية

- GraphQL — Schemas and types: https://graphql.org/learn/schema/
- NestJS — Quick start: https://docs.nestjs.com/graphql/quick-start
