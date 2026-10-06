# @nestjs/graphql

## الهدف من هذا الدرس

تشغيل مسار `/graphql` بمحرك Apollo داخل Nest، وفتح GraphiQL في التطوير، قبل كتابة أول محلل.

## المشكلة التي يحلها

معرفة SDL لا تربط HTTP بمشغّل GraphQL. من غير وحدة، لا يوجد مسار يستقبل `{ "query": "..." }`. المقالات القديمة تثبّت `apollo-server-express` و `playground`. ذلك مسار أزيل من تكامل Nest الحالي.

## الفكرة ببساطة

ثبّت حزم Apollo الخاصة بـ Express، واستورد `GraphQLModule.forRoot` مع `ApolloDriver`. واجهة التجربة اسمها GraphiQL وتُفتح خارج الإنتاج من غير خيار إضافي. خيار `playground` اسم قديم لم يعد طريقة الكتابة.

## التعريف التقني

على المحول الافتراضي Express، من صفحة البداية:

```bash
npm i @nestjs/graphql @nestjs/apollo @apollo/server @as-integrations/express5 graphql
```

Fastify مع Apollo يستبدل تكامل Express بحزمة `@as-integrations/fastify`. Mercurius حزمة `@nestjs/mercurius` ومسار آخر. هذه الدورة تبقى على Express و Apollo.

```ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      graphiql: false,
    }),
  ],
})
export class AppModule {}
```

`driver` مطلوب. الخيارات تُمرر إلى المحرك. GraphiQL هو IDE المدمج منذ `@nestjs/graphql` v14، ويُفعَّل تلقائيًا عندما لا تكون `NODE_ENV` هي `production`. `graphiql: false` يخفيه. `graphiql: true` يبقيه حتى في الإنتاج، وذلك قرار متعمد لا افتراض.

تحذير الصفحة: `playground` ما زال اسمًا منطقيًا قديمًا كقيمة بوليانية فقط، والكتابة الجديدة `graphiql`. لا تنسخ كائن إعداد Playground من مقال.

`context` يبني كائن التنفيذ لكل طلب. مثال الصفحة: `context: ({ req, res }) => ({ req, res })`. المحلل يقرؤه لاحقًا بـ `@Context()`.

## Mental Model

```text
POST /graphql
  ApolloDriver
    يتحقق من المخطط
    يستدعي المحللات المسجّلة
GET /graphql في التطوير
  GraphiQL
```

من غير `autoSchemaFile` أو `typePaths` لا يوجد مخطط بعد. ذلك درسا code-first و schema-first. هذا الدرس يوصل الأنبوب فقط.

## أبسط مثال

وحدة الجذر في التطوير، بلا إجبار الواجهة في الإنتاج:

```ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
    }),
  ],
})
export class AppModule {}
```

شغّل التطبيق وافتح `http://localhost:3000/graphql` وأنت خارج `production`. الواجهة تظهر بعد أن يوجد مخطط ومحلل، كما تقول الصفحة. قبل ذلك المسار قد يجيب بلا حقول مفيدة.

## شرح المثال

ترك `graphiql` غير مضبوط يتبع افتراض Nest: واجهة في التطوير، وصفحة هبوط مطفأة في الإنتاج حتى لا ينكشف التصفح على بيئة عامة. إن كانت بيئة تجريبية عامة و `NODE_ENV` ليست `production`، ضع `graphiql: false` صراحة.

## ماذا يحدث داخليًا؟

`forRoot` يبني الخادم ويربطه بمحول HTTP على المسار الافتراضي `/graphql`. طلب `POST` يحمل العملية. طلب المتصفح `GET` على العنوان نفسه يخدم GraphiQL عندما تكون مفعّلة. الاستبطان (introspection) هو ما تغذي به الواجهة قائمة الحقول. إطفاؤه في الإنتاج قرار أمني إن كان المخطط خاصًا. لا تضع ترويسة `Authorization` ثابتة داخل مستودع عبر `graphiql: { headers }` إلا لقيمة تطوير محلية، والخيار موجود في الصفحة لتجربة الواجهة لا لأسرار الإنتاج.

## مثال عملي

تطوير محلي مع ترويسة جاهزة للتجربة، والقيمة من بيئة المطور لا من الالتزام:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  graphiql: process.env.NODE_ENV === 'production' ? false : { shouldPersistHeaders: true },
});
```

الإنتاج يُغلق الواجهة. التطوير يبقيها ويحفظ ترويسات المحرر في المتصفح حتى لا تعيد لصق الرمز في كل تحديث.

## أخطاء شائعة

- تثبيت `apollo-server-express` من درس قديم.
- `playground: { settings: ... }` كأنه ما زال مدعومًا ككائن.
- نسيان `@as-integrations/express5` على Express 5.
- فتح GraphiQL على إنتاج عام لأنه «مفيد للدعم».

## متى نستخدمه؟

عند إضافة أول وحدة GraphQL إلى تطبيق Nest على Express. Mercurius فقط إن كان المحول Fastify وقرر الفريق ذلك صراحة.

## متى لا نستخدمه؟

لا تستدعِ `forRoot` في كل وحدة ميزة. التسجيل مرة في الجذر أو في وحدة GraphQL مستوردة مرة واحدة. المحللات تعيش في وحدات الميزات بعد أن يوجد المخطط.

## مقارنة

| الاسم في المقالات القديمة | الكتابة الحالية |
| --- | --- |
| `apollo-server-express` | `@apollo/server` مع `@nestjs/apollo` |
| `playground: true` | الافتراض خارج الإنتاج، أو `graphiql` |
| بلا `driver` | `driver: ApolloDriver` |

## تمارين

### تمرين سهل

أي خيار يسمي IDE الحالية؟

<details>
<summary>الحل</summary>
`graphiql`. Playground أزيل، والاسم القديم ليس كتابة جديدة.
</details>

### تمرين متوسط

لماذا قد تظهر الواجهة محليًا وتختفي في الإنتاج بلا أن تكتب `graphiql`؟

<details>
<summary>الحل</summary>
لأن التفعيل التلقائي مشروط بأن `NODE_ENV` ليست `production`.
</details>

### تمرين متقدم

بيئة staging عامة و `NODE_ENV=staging`. ماذا تضبط؟

<details>
<summary>الحل</summary>
`graphiql: false` إن كان المخطط لا يجب أن يُستعرض علنًا. الافتراض لا يحمي إلا قيمة `production`.
</details>

## أسئلة مراجعة

- ما حزم Express + Apollo في صفحة البداية؟
- أين يُفتح GraphiQL؟
- ماذا ينقص هذا الإعداد حتى توجد حقول؟

## الخلاصة

`GraphQLModule` مع `ApolloDriver` يفتح `/graphql`. GraphiQL واجهة التطوير. المخطط يأتي من code-first أو schema-first في الدرسين التاليين.

## ماذا بعد؟

`autoSchemaFile`: المخطط يُولَّد من أصناف TypeScript، وهو المسار الذي تكمله الدورة.

## المصادر الرسمية

- NestJS — Quick start: https://docs.nestjs.com/graphql/quick-start
