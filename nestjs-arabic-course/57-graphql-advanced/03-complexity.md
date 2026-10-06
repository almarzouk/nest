# Complexity

## الهدف من هذا الدرس

رفض استعلام كتب رف الذي تتجاوز تكلفته سقفًا تحسبه قبل التنفيذ، باستخدام `graphql-query-complexity` كما تدمجه صفحة Complexity في Nest، لا بديكور خيالي اسمه `@MaxComplexity`.

## المشكلة التي يحلها

بعد إصلاح N+1 ما زال العميل يستطيع طلب `books(limit: 500)` مع حقول علاقتها غالية. كل حقل رخيص وحده. المجموع يضغط القاعدة. التحقق من العمق لا يكفي لأن قائمة عريضة وضحلة قد تكون أغلى من سلسلة قصيرة.

## الفكرة ببساطة

كل حقل له رقم تكلفة. تُجمع الأرقام على المستند بعد ما يُفهم وقبل ما يُنفَّذ. إن تجاوز المجموع السقف تُرمى `GraphQLError` ولا تُستدعى الخدمات. الحقل الغالي يصرّح بتكلفته. الباقي يأخذ تكلفة افتراضية.

## التعريف التقني

الصفحة رسمية للنهج code first. الحزمة `graphql-query-complexity` توفّر `getComplexity` و `fieldExtensionsEstimator` و `simpleEstimator`. تضع الصنف الذي ينفّذ `ApolloServerPlugin` وتوسمه بـ `@Plugin()` من `@nestjs/apollo`، وتحقن `GraphQLSchemaHost` من `@nestjs/graphql`. داخل `didResolveOperation` تحسب التعقيد من `schema` و `document` و `variables`. التقدير الأول الذي يعيد رقمًا يفوز. `fieldExtensionsEstimator` يقرأ `complexity` من `@Field` و `@Query`. `simpleEstimator({ defaultComplexity: 1 })` يغطي الباقي. السقف في مثال التوثيق 20 للعرض؛ رف تختار سقفها بالقياس. الصنف يُضاف إلى `providers`. لا يشحن Nest هذه القاعدة كخيار `complexity: 20` على `GraphQLModule` وحده.

## Mental Model

```text
books(limit) تكلفة = limit × تكلفة الابن
author تكلفة 1 عبر simpleEstimator
المجموع > السقف → GraphQLError قبل الخدمة
```

## أبسط مثال

```ts
import { Field, Int, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field(() => Int, { complexity: 1 })
  priceCents!: number;
}
```

## شرح المثال

- `complexity: 1` يذهب إلى extensions ويقرأه `fieldExtensionsEstimator`.
- من غير الإضافة في خيارات الحقل، `simpleEstimator` هو الذي يضع الرقم الافتراضي.
- الحقل لا يرفض الاستعلام وحده. الجمع والسقف في الإضافة.

## ماذا يحدث داخليًا؟

1. Apollo يحل المستند ويصل إلى `didResolveOperation`.
2. `getComplexity` يمشي على الشجرة ويستدعي المقدّرات بالترتيب.
3. حقل فيه `complexity` رقمًا أو دالة يأخذ ذلك الرقم.
4. دالة على `@Query` مثل `options.args.limit * options.childComplexity` تجعل الحد يضاعف تكلفة الأبناء.
5. فوق السقف تُرمى `GraphQLError` فلا يعمل `@Query` الذي يضرب Prisma.

## مثال عملي

تكلفة قائمة تتناسب مع `limit`، كما في صفحة Complexity:

```ts
import { Args, Int, Query, Resolver } from '@nestjs/graphql';
import { Book } from './book.model.js';

@Resolver()
export class BooksResolver {
  @Query(() => [Book], {
    complexity: (options) => Number(options.args.limit ?? 20) * (options.childComplexity ?? 1),
  })
  books(@Args('limit', { type: () => Int, nullable: true }) limit = 20) {
    return [];
  }
}
```

الإضافة نفسها من التوثيق: `maxComplexity` الذي اخترتموه، والمقدّران `fieldExtensionsEstimator()` ثم `simpleEstimator({ defaultComplexity: 1 })`. `limit` غير المقيّد في الـ Pipe ما زال خطرًا؛ التعقيد خط دفاع ثانٍ بعد حد 50 من عقد REST إن شاركتم نفس الخدمة.

## أخطاء شائعة

- سقف 20 منسوخ من مثال العرض على مخطط إنتاج فيرفض `books { title author { name } }` العادي.
- حساب التعقيد بعد جلب الصفوف. الفائدة أن الرفض يسبق الخدمة.
- نسيان تسجيل الصنف في `providers` فتبقى الدالة بلا أثر.
- اعتبار التعقيد بديلًا عن DataLoader. استعلام تحت السقف ما زال يستطيع عمل N+1.

## متى نستخدمه؟

مخطط عام يفتح حقولًا متداخلة وحدود قوائم يختارها العميل.

## متى لا نستخدمه؟

مخطط داخلي بعمليات معروفة وثابتة. هناك قائمة استعلامات محفوظة أقوى، ودرسها لاحقًا. ولا كبديل لاختبار حمل حقيقي.

## مقارنة

| الأداة | ماذا تحد |
| --- | --- |
| DataLoader | عدد الرحلات لكل مستوى |
| Complexity | مجموع تكلفة الشجرة قبل التنفيذ |
| حد `limit` في الـ Args | عرض قائمة واحدة |
| Depth | طول السلسلة فقط |

## تمارين

### تمرين سهل

حقل بلا `complexity` ومقدّر بسيط افتراضيه 1. كم يساهم الحقل؟

<details>
<summary>الحل</summary>

1، لأن `simpleEstimator` يعمل عندما لا يعيد المقدّر السابق رقمًا.

</details>

### تمرين متوسط

لماذا تُضرب تكلفة `books` في `limit` لا تُترك 1؟

<details>
<summary>الحل</summary>

لأن كل عنصر يعيد شجرة أبناء. تكلفة 1 للقائمة تخفي أن `limit: 200` أغلى بعشرين مرة من `limit: 10`. الدالة في مثال التوثيق تضرب `args` في `childComplexity`.

</details>

### تمرين متقدم

الاستعلام تحت السقف وما زال SQL ينفجر. أي درس يعود، ولماذا لا ترفع السقف أولًا؟

<details>
<summary>الحل</summary>

درس N+1 و DataLoader. رفع السقف يسمح باستعلام أغلى على حل سيئ. أصلح الرحلات ثم عدّل السقف حسب قياس المسار الشرعي.

</details>

## أسئلة مراجعة

- أي حزمة تحسب الرقم، وأي ديكور Nest يعلن الإضافة؟
- متى تُرمى `GraphQLError` نسبة إلى تنفيذ الـ Resolver؟
- ماذا يقرأ `fieldExtensionsEstimator`؟
- لماذا السقف 20 في التوثيق ليس معيار رف؟

## الخلاصة

التعقيد مجموع تكاليف الحقول قبل التنفيذ. Nest يدمجه بإضافة Apollo وحزمة `graphql-query-complexity` وخيار `complexity` على `@Field` و `@Query`. ليس خيارًا واحدًا على الوحدة يغني عن تسجيل الإضافة.

## ماذا بعد؟

التعقيد يوزن العرض. سلسلة `book { author { books { author { books } } } }` قد تبقى تحت سقف ساذج إن كانت التكاليف 1. الدرس التالي يحد العمق.

## المصادر الرسمية

- NestJS — Complexity: https://docs.nestjs.com/graphql/complexity
- graphql-query-complexity: https://github.com/slicknode/graphql-query-complexity
