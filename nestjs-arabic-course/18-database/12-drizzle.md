# Drizzle و @nestjs/drizzle

## الهدف من هذا الدرس

معرفة التكامل الرسمي لـ Drizzle: مخطط TypeScript، `DrizzleModule.forRoot`، وحقن `@InjectDrizzle()`، كبديل موثّق لا كمسار ثانٍ داخل المشروع.

## المشكلة التي يحلها

من يريد استعلامًا قريبًا من SQL مع أنواع مستنتجة قد يجد كيانات TypeORM ثقيلة والعميل المولَّد لـ Prisma خطوة توليد إضافية. Nest يوثّق `@nestjs/drizzle` لهذا الأسلوب. تجاهل الصفحة يجعل الفريق يخترع تهيئة مجمع لا تُغلق مع التطبيق.

## الفكرة ببساطة

تعرّف الجدول بدالة `pgTable`. تسجّل `DrizzleModule.forRoot` بدالة `drizzle` الخاصة بالمشغّل وسلسلة الاتصال. أي مزوّد يحقن قاعدة `NodePgDatabase` ويبني `select` و `insert`. لا `forFeature`. الجداول كائنات تستوردها حيث تكتب الاستعلام.

## التعريف التقني

من صفحة Drizzle في Nest. الحزم في تلك الصفحة تستخدم Drizzle ORM v1 على وسم `rc` وقت التوثيق، مع دعم أقدم من `0.35`:

```bash
npm install --save @nestjs/drizzle drizzle-orm@rc pg
npm install --save-dev drizzle-kit@rc @types/pg
```

```ts
import { drizzle } from "drizzle-orm/node-postgres";

DrizzleModule.forRoot({
  drizzle,
  connection: process.env.DATABASE_URL!,
});
```

الوحدة تستدعي `drizzle()` مرة لكل تطبيق، فلكل اختبار e2e مجمعه الذي يُغلق عند الإيقاف. `autoCloseConnection` افتراضيًا `true`.

خيارات `forRoot` تُقيَّم عند استيراد الملف، لذلك `DATABASE_URL` يجب أن يكون حاضرًا قبل ذلك، أو تستخدم `forRootAsync` وتقرأ `ConfigService`.

```ts
constructor(@InjectDrizzle() private readonly db: NodePgDatabase) {}
```

استعلام:

```ts
this.db.select().from(books);
this.db.insert(books).values(input).returning();
```

`eq` من `drizzle-orm` يبني الشرط. الأنواع `typeof books.$inferSelect` و `$inferInsert`.

الترحيل عبر drizzle-kit، لا عبر `synchronize`. المعاملات في الصفحة عبر واجهة Drizzle على `db.transaction`. لا تخلطها مع `$transaction` في المشروع نفسه.

MongoDB ليس هذا المسار. Drizzle هنا على PostgreSQL مثل بقية الدورة.

## Mental Model

```text
pgTable("books")  كائن مخطط
DrizzleModule.forRoot
@InjectDrizzle()  قاعدة واحدة لكل تطبيق
select/insert     SQL منشئ
```

## أبسط مثال

```ts
import { integer, pgTable, text, boolean } from "drizzle-orm/pg-core";

export const books = pgTable("books", {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  title: text().notNull(),
  published: boolean().notNull().default(false),
});

export type Book = typeof books.$inferSelect;
```

```ts
import { Injectable } from "@nestjs/common";
import { InjectDrizzle } from "@nestjs/drizzle";
import { eq } from "drizzle-orm";
import type { NodePgDatabase } from "drizzle-orm/node-postgres";
import { books } from "../db/schema.js";

@Injectable()
export class BooksService {
  constructor(@InjectDrizzle() private readonly db: NodePgDatabase) {}

  list() {
    return this.db.select().from(books);
  }

  findOne(id: number) {
    return this.db.select().from(books).where(eq(books.id, id));
  }
}
```

## شرح المثال

`books` جدول يمكن استيراده في الخدمة من غير تسجيل مزوّد لكل كيان. `eq` يربط القيمة بمعامل. `findOne` يرجع مصفوفة، فخذ العنصر الأول في الخدمة إن كان العقد كتابًا واحدًا أو غيابه.

إن كان `connection` يُقرأ من `process.env` عند الاستيراد، انقل التسجيل إلى `forRootAsync` بعد `ConfigModule` حتى لا يتكرر شرط فصل الإعداد.

## ماذا يحدث داخليًا؟

المجمع يُنشأ مع التطبيق ويُغلق عند `app.close()` أو عند إشارة الإيقاف إن فُعّلت الخطافات، باستدعاء `end()` على عملاء المجمع. مشاركة كائن `db` مُنشأ عند أول استيراد بين عدة تطبيقات اختبار تكسر الإغلاق: أول `close` يقفل الباقي. الصفحة تحذّر من ذلك وتنصح بإنشاء القاعدة داخل المصنع لكل تطبيق.

## مثال عملي

تجربة فرع قصيرة تقارن استعلام الإعارات بين Prisma و Drizzle على نسخة قاعدة تطوير. لا تُبقِ الوحدتين في `AppModule` بعد التجربة. اختر واحدة في درس الاختيار. ترحيلات drizzle-kit تبقى مجلد SQL خاصًا بها، فلا تخلطها مع مجلد `prisma/migrations` على القاعدة نفسها من غير خطة.

## أخطاء شائعة

- تمرير كائن `db` مشترك لكل تطبيقات الاختبار مع إغلاق تلقائي.
- لصق مدخل المستخدم في `sql.raw`.
- تشغيل Drizzle مع Prisma على الجداول نفسها في النسخة النهائية.
- نسيان أن خيارات `forRoot` المبكرة لا ترى `.env` إن لم يُحمَّل الملف قبل الاستيراد.

## متى نستخدمه؟

فريق يريد مخططًا في TypeScript واستعلامًا ظاهرًا، مع تكامل Nest الذي يغلق المجمع.

## متى لا نستخدمه؟

لإضافة أداة ثالثة بجانب Prisma الذي بنيت عليه خدمات الكتب والإعارات في هذا الفصل.

## مقارنة

| Prisma | Drizzle عبر Nest |
| --- | --- |
| مخطط واحد وعميل مولَّد | `pgTable` وأنواع مستنتجة |
| `PrismaService` تمتد العميل | `@InjectDrizzle()` |
| ترحيل Prisma | drizzle-kit |
| مسار الدورة | بديل موثّق |

## تمارين

### تمرين سهل

أي مزخرف يحقن قاعدة Drizzle؟

<details>
<summary>الحل</summary>

`@InjectDrizzle()` من `@nestjs/drizzle`.

</details>

### تمرين متوسط

لماذا لكل تطبيق اختبار مجمعه الخاص في هذا التكامل؟

<details>
<summary>الحل</summary>

`DrizzleModule` يستدعي `drizzle()` مرة لكل تطبيق ويغلق المجمع عند إيقاف ذلك التطبيق. مشاركة عميل واحد تجعل أول إيقاف يغلق القاعدة على الاختبارات الأخرى.

</details>

### تمرين متقدم

`forRoot({ connection: process.env.DATABASE_URL })` يرى `undefined` رغم `ConfigModule`. ما التعديل؟

<details>
<summary>الحل</summary>

الخيار يُحسب عند الاستيراد. استخدم `DrizzleModule.forRootAsync` واقرأ العنوان من `ConfigService` داخل `useFactory`، أو حمّل البيئة قبل الإقلاع بعلم `--env-file`.

</details>

## أسئلة مراجعة

- هل يوجد `forFeature` في هذا التكامل؟
- من يغلق المجمع؟
- لماذا لا نثبّته مع Prisma في المكتبة النهائية؟

## الخلاصة

`@nestjs/drizzle` يسجّل قاعدة لكل تطبيق، والاستعلام منشئ على جداول TypeScript. هو البديل الموثّق الثاني بعد TypeORM، لا طبقة إضافية فوق Prisma.

## ماذا بعد؟

[متى تختار Prisma أو TypeORM أو Drizzle](./13-choosing-a-data-tool.md) يغلق الفصل بقرار واحد.

## المصادر الرسمية

- NestJS — Drizzle: https://docs.nestjs.com/data/drizzle
- NestJS — Databases: https://docs.nestjs.com/data/overview
