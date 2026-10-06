# أساسيات Federation

## الهدف من هذا الدرس

فهم كيف يتوزع نوع `Book` بين خدمة كتالوج وخدمة مراجعات في رف، عبر بوابة ومخططات فرعية، باستخدام ما توثقه صفحة Federation: `@key` و `resolveReference` و `ApolloFederationDriver`.

## المشكلة التي يحلها

فريق المراجعات يريد نشر حقول تقييم الكتاب من غير انتظار نشر الكتالوج. دمج المخططات يدويًا في ملف SDL واحد يعيد الاقتران الذي أرادوا كسره. العميل ما زال يريد استعلامًا واحدًا: عنوان الكتاب وآخر مراجعة.

## الفكرة ببساطة

كل خدمة تملك جزءًا من النوع وتعلن مفتاحًا يجد به الكيان. البوابة تدمج المخططات وتسأل الخدمة المالكة عندما يحتاج حقلًا من خدمة أخرى. Nest في الخدمة الفرعية يشغّل المحرك الاتحادي. البوابة طبقة أخرى. هذا ليس `@Federation()` على `BooksModule` ينهي القصة.

## التعريف التقني

الصفحة تصف مكوّنين: gateway وخدمات متحدة. المبادئ من Apollo: التكوين تصريحي، والنوع الواحد قد يتوزع على فرق، والعميل يرى مخططًا واحدًا، والأدوات ضمن مواصفات GraphQL. الاشتراكات غير مدعومة في هذا الأسلوب حسب تحذير الصفحة. على الخدمة: ثبّت `@apollo/subgraph`. في النهج schema first، `type Book @key(fields: "id")` يعلن المفتاح. الدالة `resolveReference` مُعلَّمة بـ `@ResolveReference()` من `@nestjs/graphql` وتعيد الكيان عندما تسأل البوابة عنه. `GraphQLModule.forRoot` يستخدم `ApolloFederationDriver` و `ApolloFederationDriverConfig` من `@nestjs/apollo`. البوابة تركّب المخططات؛ حقول المراجعات تعيش في خدمتها وتشير إلى `Book` بالمفتاح.

## Mental Model

```text
عميل → بوابة → كتالوج: Book.id + title
                 مراجعات: Book.reviews عبر id
resolveReference في الكتالوج يعيد الكتاب عندما تأتي المراجعات بمفتاح
```

## أبسط مثال

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class Book {
  @Field(() => ID)
  id!: string;
}
```

في schema first يكون نفس الإعلان داخل SDL كما في الصفحة. في code first، `@Directive('@key(fields: "id")')` يضع المفتاح على النوع. اختر نهجًا واحدًا للخدمة.

## شرح المثال

- `id` هو ما ترسله البوابة لتطلب هذا الكتاب من مالكه.
- الحقول التي لا تملكها الخدمة لا تُنسخ إليها «للراحة».
- `@Resolver` يحتاج `resolveReference` إن كانت الخدمة مرجعًا يُسأل بالمفتاح.

## ماذا يحدث داخليًا؟

1. البوابة تجلب مخطط كل خدمة.
2. استعلام العميل يُقسَّم إلى أجزاء حسب مالك الحقل.
3. عندما تحتاج المراجعات كتابًا، البوابة تستدعي `_entities` ومعه `{ __typename: "Book", id }`.
4. `resolveReference` في الكتالوج تستدعي `books.findById(reference.id)`.
5. البوابة تجمع الأجزاء في رد واحد. فشل الكتالوج يفشل الحقول التي تعتمد عليه.

## مثال عملي

```ts
import { ResolveReference, Resolver } from '@nestjs/graphql';
import { BooksService } from './books.service.js';

@Resolver('Book')
export class BooksFederationResolver {
  constructor(private readonly books: BooksService) {}

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.books.findById(reference.id);
  }
}
```

التسجيل من الصفحة:

```ts
import { ApolloFederationDriver, type ApolloFederationDriverConfig } from '@nestjs/apollo';
import { GraphQLModule } from '@nestjs/graphql';

GraphQLModule.forRoot<ApolloFederationDriverConfig>({
  driver: ApolloFederationDriver,
  typePaths: ['**/*.graphql'],
});
```

`typePaths` للنهج schema first كما تبدأ الصفحة. البوابة ليست هذا السطر؛ هي عملية تركّب عناوين الخدمات. DataLoader ما زال لازمًا داخل `resolveReference` إن سألت البوابة عن عشرات الكتب دفعة واحدة، وإلا عاد N+1 عبر حدود الخدمة.

## أخطاء شائعة

- نسخ نوع `Book` كاملًا في كل خدمة فتتباعد الحقول.
- بوابة بلا `resolveReference` في المالك، فالحقول الممتدة فارغة أو تفشل.
- توقع subscriptions عبر Federation. الصفحة تقول إنها غير مدعومة حاليًا.
- تقديم الاتحاد قبل أن يثبت مخطط العملية الواحدة. التوزيع يضيف شبكة لا يلغي تعقيد الاستعلام.

## متى نستخدمه؟

عندما تمتلك فرقًا مختلفة حقولًا مختلفة على نفس الكيان وتريد نشرًا مستقلًا وعميلًا باستعلام واحد.

## متى لا نستخدمه؟

لمتجر رف في عملية واحدة بفريق واحد. المخطط الواحد أبسط. ولا كبديل عن حدود REST إن كان العميل أصلًا لا يستخدم GraphQL.

## مقارنة

| | مخطط واحد | Federation |
| --- | --- | --- |
| النشر | معًا | كل خدمة على حدة |
| استعلام العميل | مباشر | عبر البوابة |
| ما يضاف في Nest | `GraphQLModule` المعتاد | `ApolloFederationDriver` و `@ResolveReference` و `@apollo/subgraph` |
| الاشتراكات | حسب إعدادك | غير مدعومة هنا حسب الصفحة |

## تمارين

### تمرين سهل

من يجيب عن `title` ومن يجيب عن `reviews` إن كان كل فريق يملك حقله؟

<details>
<summary>الحل</summary>

الكتالوج يجيب عن `title` عبر كيانه. المراجعات تجيب عن `reviews`. البوابة تسأل الكتالوج بـ `resolveReference` عندما تحتاج كيان `Book` من مفتاح `id`.

</details>

### تمرين متوسط

لماذا DataLoader داخل الخدمة المتحدة ما زال واردًا؟

<details>
<summary>الحل</summary>

البوابة قد تطلب عشرات الكيانات في عملية `_entities` واحدة. `findById` لكل مرجع يعيد N+1 داخل الكتالوج. مجمّع الطلب يجمع المعرّفات.

</details>

### تمرين متقدم

اذكر مكوّنين من التوثيق لا يغني عنهما ديكور على `BooksModule`.

<details>
<summary>الحل</summary>

البوابة التي تدمج المخططات، وحزمة `@apollo/subgraph` مع المحرك `ApolloFederationDriver`. الديكور `@ResolveReference` جزء من الخدمة فقط. من غيره تعرف البوابة المفتاح ولا تجد من يعيد الكيان.

</details>

## أسئلة مراجعة

- ماذا يعلن `@key`؟
- متى تُستدعى `resolveReference`؟
- أي تحذير بشأن الاشتراكات؟
- لماذا الاتحاد ليس خطوة الأداء الأولى؟

## الخلاصة

Federation بوابة وخدمات. الخدمة في Nest تصرّح بالمفتاح وتنفّذ `resolveReference` على `ApolloFederationDriver`. العميل يرى مخططًا واحدًا. لا ديكور واحد في الوحدة يستبدل البوابة ولا يلغي N+1 داخل الخدمة.

## ماذا بعد؟

المخطط المحمي ما زال يحتاج هوية. الفصل التالي يوسّع المصادقة إلى OAuth و OpenID Connect وتدوير الرمز وإبطاله.

## المصادر الرسمية

- NestJS — Federation: https://docs.nestjs.com/graphql/federation
- Apollo — Federation: https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/federation
