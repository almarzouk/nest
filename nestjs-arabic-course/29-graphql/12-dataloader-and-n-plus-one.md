# DataLoader و مشكلة N+1

## الهدف من هذا الدرس

تسمية مشكلة N+1 ورؤية أن DataLoader يُنشأ لكل طلب على context. التحميل المجمّع الكامل مكانه فصل لاحق.

## المشكلة التي يحلها

`books { author { name } }` على عشرين كتابًا يشغّل `@ResolveField` عشرين مرة. إن كانت كل مرة `SELECT` بمعرف، صار الاستعلام 1 للقائمة + N للمؤلفين. البطاقة التي لا تطلب `author` لا تدفع N. التفاصيل التي تطلبها تدفعها. إخفاء المشكلة بجلب كل المؤلفين في `books()` يهدر عندما لا يُطلب الحقل.

## الفكرة ببساطة

DataLoader، من مكتبة `dataloader`، يجمع المفاتيح التي تُطلب في الدورة نفسها وينادي دالة دفعة واحدة. في GraphQL الدورة هي تنفيذ العملية. ضع المحمّل على `context` عند بناء الطلب، لا كمزوّد singleton، وإلا اختلطت مفاتيح المستخدمين وبقيت ذاكرة التخزين المؤقت داخل المحمّل عبر الطلبات.

هذا الدرس لا يبني طبقة مستودع ولا يقيس Redis. هو يمنع المحمّل العام الخطأ. الفصل `57-graphql-advanced` يعيد DataLoader بعمق.

## التعريف التقني

الشكل الذي يكفي هنا:

```ts
import DataLoader from 'dataloader';

export function createAuthorLoader(authors: AuthorsService) {
  return new DataLoader<string, { id: string; name: string }>(async (ids) => {
    const rows = await authors.findManyByIds(ids);
    const byId = new Map(rows.map((row) => [row.id, row]));
    return ids.map((id) => byId.get(id) ?? new Error(`Author ${id} missing`));
  });
}
```

الدالة تعيد مصفوفة بالترتيب نفسه لمصفوفة `ids`. هذا عقد DataLoader، لا تفصيل Nest.

في الإعداد:

```ts
context: ({ req, res }) => ({
  req,
  res,
  authorLoader: createAuthorLoader(authorsService),
}),
```

حقن الخدمة داخل `context` في `forRoot` يحتاج أن تكون الخدمة متاحة هناك، غالبًا عبر `GraphQLModule.forRootAsync`. لا تحشر `new AuthorsService()` بلا الحاوية. إن صعب الربط الآن، أبقِ `@ResolveField` كما في درس العلاقات حتى فصل 57، وتذكّر أن singleton loader هو الخطأ الذي ستتجنبه.

## Mental Model

```text
لكل POST /graphql
  authorLoader جديد
    author(book1) load(id)
    author(book2) load(id)
      دالة واحدة findManyByIds
الطلب التالي محمّل فارغ
```

## أبسط مثال

المحلل بعد أن يوجد المحمّل على context:

```ts
@ResolveField(() => Author)
author(
  @Parent() book: { authorId: string },
  @Context('authorLoader') loader: DataLoader<string, Author>,
) {
  return loader.load(book.authorId);
}
```

عشرون `load` في التنفيذ نفسه تصبح نداء `findManyByIds` واحدًا إن جُمعت في الدورة.

## شرح المثال

`load` يعيد وعدًا. DataLoader ينتظر نهاية الدورة المتزامنة ثم يجمع المفاتيح الفريدة. تكرار معرف المؤلف في كتب عدة لا يكرر العنصر في الدفعة. الناتج يُوزَّع على كل `load`. الطلب التالي لا يرى ذاكرة الطلب السابق لأن الدالة `context` بنت محمّلًا جديدًا.

## ماذا يحدث داخليًا؟

من غير محمّل، حل الحقول يشغّل الدالة لكل أب فورًا. مع المحمّل، الاستدعاءات تُجدول. إن كانت `findManyByIds` ما زالت تعمل حلقة استعلام داخلها، لم تربح شيئًا. الدفعة يجب أن تكون استعلام `IN` أو ما يعادله. DataLoader لا يخترع SQL.

ذاكرة المحمّل داخل الطلب مفيدة لتكرار المعرف. بقاؤها على مزوّد التطبيق تخدم مستخدمًا آخر بيانات قد لا يجوز أن يراها، وتُقدم اسم مؤلف تغيّر في الصف. لذلك الربط بالطلب جزء من الصحة لا من الأداء فقط.

## مثال عملي

قياس قبل المكتبة: سجّل عدد نداءات `findById` للمؤلف أثناء `query { books { author { name } } }`. إن رأيت N، فالمشكلة قائمة. بعد المحمّل لكل طلب يجب أن ترى نداء دفعة. إن بقي N، فالدفعة ليست مجمّعة. أوقف التوسع في الواجهة عند هذا الاختبار. تفاصيل `forRootAsync` وحقن Prisma تُكتب في الفصل 57 لا هنا.

## أخطاء شائعة

- `providers: [AuthorLoader]` كـ singleton للتطبيق.
- إرجاع صفوف `findManyByIds` بترتيب القاعدة لا بترتيب `ids`.
- جلب المؤلف في `books()` دائمًا وتعطيل فائدة «لا تطلب الحقل».
- اعتبار هذا الدرس بديلًا عن فهرس المعرف في الجدول.

## متى نستخدمه؟

عندما يضرب `@ResolveField` مصدرًا لكل عنصر في قائمة يطلبها العميل فعلًا.

## متى لا نستخدمه؟

قبل أن ترى N في السجل. محمّل لكل حقل من اليوم الأول يخفي تكلفة لم تقيسها. وليس مكانه كاش HTTP عام بين المستخدمين.

## مقارنة

| الأسلوب | استعلامات مؤلف لعشرين كتابًا |
| --- | --- |
| `@ResolveField` يستعلم كل مرة | 1 + 20 |
| DataLoader لكل طلب | 1 + دفعة |
| `include` دائم في الأب | 1 حتى لو لم يُطلب `author` |

## تمارين

### تمرين سهل

ماذا يعني N في N+1 هنا؟

<details>
<summary>الحل</summary>

نداءً إضافيًا لكل كتاب لجلب مؤلفه، بعد نداء القائمة.

</details>

### تمرين متوسط

لماذا يُنشأ المحمّل داخل `context`؟

<details>
<summary>الحل</summary>

حتى تُجمع مفاتيح هذه العملية فقط، ولا تبقى النتائج لمستخدم أو طلب لاحق.

</details>

### تمرين متقدم

الدفعة أعادت الصفوف مرتبة بالاسم لا بترتيب `ids`. ماذا يرى `load`؟

<details>
<summary>الحل</summary>

مؤلفًا غير مطابق للمفتاح. عقد DataLoader يفرض ناتجًا بمحاذاة مصفوفة المدخلات.

</details>

## أسئلة مراجعة

- متى لا تُدفع تكلفة N؟
- ما عقد ترتيب الدالة الدفعية؟
- أي فصل يعمّق DataLoader؟

## الخلاصة

N+1 تظهر عندما يطلب العميل حقل علاقة على قائمة. DataLoader لكل طلب يجمع المفاتيح. لا تبنِ محمّلًا عامًا، واترك التفصيل للفصل `57-graphql-advanced`.

## ماذا بعد؟

اشتراك يبقى مفتوحًا: `graphql-ws` وحقل على جذر `Subscription`.

## المصادر الرسمية

- GraphQL — DataLoader: https://github.com/graphql/dataloader
- NestJS — Quick start: https://docs.nestjs.com/graphql/quick-start
