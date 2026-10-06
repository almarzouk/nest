# Code First

## الهدف من هذا الدرس

جعل أصناف TypeScript مصدر المخطط: Nest تكتب SDL من الزينة إلى ملف أو إلى الذاكرة.

## المشكلة التي يحلها

ملف `.graphql` وصنف TypeScript يصفان الكتاب مرتين. يعدّل الفريق العنوان في واحد وينسى الآخر، فيفشل التشغيل أو يكذب التوثيق. Code-first يبقي الزينة على الصنف، ويولّد SDL للعين وللأدوات.

## الفكرة ببساطة

تضبط `autoSchemaFile`. كل `@ObjectType` و `@InputType` و `@Resolver` مسجّل في شجرة الوحدات يدخل التوليد. المسار الذي تكمله هذه الدورة هو هذا. Schema-first في الدرس التالي حتى تعرف الباب الآخر، لا لتكتبه بالتوازي على الميزة نفسها.

## التعريف التقني

من صفحة البداية:

```ts
import { join } from 'node:path';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { GraphQLModule } from '@nestjs/graphql';

GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,
});
```

`autoSchemaFile: true` يولّد المخطط في الذاكرة بلا ملف. مفيد للاختبار. الملف المفيد في التطوير لأن فرق Git يُظهر تغيير العقد.

`sortSchema: true` يرتب الأنواع معجميًا. من غيره يكون الترتيب ترتيب التعريف في الوحدات، فيتغير الفرق عند تحريك ملف.

الزينة التي ستكتبها لاحقًا (`@ObjectType` و `@Field` و `@Resolver`) هي مدخلات هذا التوليد. هذا الدرس لا يشرح كل زينة. هو يشرح متى يُعاد بناء `schema.gql`.

لا تجمع `autoSchemaFile` مع `typePaths` على التطبيق نفسه. مصدر واحد للحقيقة.

## Mental Model

```text
صنف Book + زينة
  وحدات مستوردة من الجذر
    GraphQLModule يمسحها عند الإقلاع
      schema.gql أو مخطط في الذاكرة
        Apollo ينفذ العمليات ضدها
```

صنف غير مسجّل في وحدة واصلة للجذر لا يظهر في الملف.

## أبسط مثال

جذر التطبيق بعد تثبيت الحزم:

```ts
import { Module } from '@nestjs/common';
import { join } from 'node:path';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { GraphQLModule } from '@nestjs/graphql';
import { BooksModule } from './books/books.module.js';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
      sortSchema: true,
    }),
    BooksModule,
  ],
})
export class AppModule {}
```

شغّل الخادم. إن كان `BooksModule` يصدّر محللًا ونوعًا، يتكون `src/schema.gql`. افتحه واقرأ `type Query` قبل فتح GraphiQL.

## شرح المثال

`join(process.cwd(), ...)` يكتب الملف من مجلد التشغيل، فيجده Git. لا تضع الملف تحت `dist` فقط ثم تتساءل لماذا الفرق فارغ. `BooksModule` في `imports` حتى تدخل أصنافه المسح. توليد الملف أثر جانبي للإقلاع: أعد التشغيل بعد تغيير الزينة.

## ماذا يحدث داخليًا؟

عند الإقلاع تجمع Nest البيانات الوصفية للأنواع والمحللات المسجّلة وتبني مخطط GraphQL. ثم تكتبه إن كان `autoSchemaFile` مسارًا. الطلب التالي يُتحقق منه ضد هذا المخطط لا ضد نسخة حفظتها يدويًا. إن حذفت حقلًا من الصنف وما زال العميل القديم يطلبه، فشل التحقق بعد إعادة التشغيل. لذلك الملف المولَّد يستحق مراجعة في طلب الدمج كأي عقد عام.

## مثال عملي

في الاختبارات لا تحتاج ملفًا:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
  sortSchema: true,
});
```

نفس الأصناف، بلا كتابة على القرص أثناء `vitest` أو `jest`. في التطبيق الإقلاعي أبقِ المسار ملفًا حتى يقرأ الإنسان العقد.

## أخطاء شائعة

- تعديل `schema.gql` باليد ثم فقدان التعديل في الإقلاع التالي.
- نسيان استيراد وحدة الميزة، فيُولَّد مخطط بلا `book`.
- تشغيل code-first و schema-first معًا على الجذر نفسه.
- البحث عن الحقل في GraphiQL قبل إعادة التشغيل.

## متى نستخدمه؟

لمسار هذه الدورة: فريق يكتب TypeScript ويريد SDL ناتجًا لا مصدرًا.

## متى لا نستخدمه؟

عندما يُنشر المخطط كعقد تملكه فرق غير TypeScript وتسلّمه لك كملفات SDL. عندها schema-first مصدرهم، لا ملفًا تولّده ثم تعدّله.

## مقارنة

| | Code first | تحرير `schema.gql` يدويًا |
| --- | --- | --- |
| المصدر | الأصناف | الملف، ويُستبدل عند التوليد |
| فرق Git | يتغير عندما تتغير الزينة | يتعارك مع التوليد |
| في هذه الدورة | المسار المعتمد | لا |

## تمارين

### تمرين سهل

أي خيار يكتب المخطط إلى `src/schema.gql`؟

<details>
<summary>الحل</summary>
`autoSchemaFile: join(process.cwd(), 'src/schema.gql')`.
</details>

### تمرين متوسط

ماذا يغيّر `sortSchema: true` في مراجعة الشيفرة؟

<details>
<summary>الحل</summary>
يثبّت ترتيب الأنواع في الملف، فلا يظهر فرق لمجرد إعادة ترتيب الاستيراد.
</details>

### تمرين متقدم

لماذا `autoSchemaFile: true` أنسب لاختبار الوحدة من مسار ملف؟

<details>
<summary>الحل</summary>
المخطط يُبنى في الذاكرة. الاختبار لا يكتب `schema.gql` ولا يصطدم بتشغيل آخر على القرص.
</details>

## أسئلة مراجعة

- ما مصدر الحقيقة في code-first؟
- لماذا يجب أن تُستورد وحدة الكتب؟
- أي خيار تتجنبه على التطبيق نفسه لاحقًا؟

## الخلاصة

`autoSchemaFile` يولّد SDL من زينة الأصناف المسجّلة. الملف للمراجعة، والذاكرة للاختبار، والتعديل اليدوي على الناتج يضيع.

## ماذا بعد؟

Schema-first: عندما تكون ملفات SDL هي المصدر، وNest تولّد أنواع TypeScript منها.

## المصادر الرسمية

- NestJS — Quick start (Code first): https://docs.nestjs.com/graphql/quick-start
