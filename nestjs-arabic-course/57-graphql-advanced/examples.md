# أمثلة عملية

استعلام شاشة كتاب في رف، ثم ما يحيط به من غير خلط الأدوات في ديكور واحد.

## المثال 1 — الحقل والمجمّع

```ts
@ResolveField()
author(@Parent() book: { authorId: string }) {
  return this.authorLoader.loader.load(book.authorId);
}
```

`authorLoader` نطاقه الطلب، و `batch` تعيد الصفوف بترتيب المفاتيح وبقيد المستأجر. اختبار القائمة يعد رحلة مؤلف واحدة لا N.

## المثال 2 — رفض قبل الخدمة

شغّل قاعدتين على المستند الذي تعرفه الواجهة:

- عمق يسمح بـ `book → reviews → user` ويرفض إعادة الدخول إلى `books` ست مرات.
- تعقيد يضرب `limit` في تكلفة الأبناء عبر `graphql-query-complexity` وإضافة `@Plugin()` كما في صفحة Complexity.

ثم قائمة مغلقة: تجزئة `query Shelf { books { title } }` مسجّلة من البناء، وتجزئة عشوائية ترجع `Persisted query not found` ولا تصل إلى Prisma.

## المثال 3 — مفتاح الاتحاد فقط

```graphql
type Book @key(fields: "id") {
  id: ID!
  title: String!
}
```

`resolveReference` في الكتالوج تستدعي `findById`. حقل `reviews` يبقى في خدمة المراجعات. البوابة خارج هذا الملف. لا تضع `@ResolveField` للمراجعات داخل الكتالوج نسخةً ثانية من البيانات.
