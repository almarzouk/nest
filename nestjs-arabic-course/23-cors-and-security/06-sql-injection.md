# SQL Injection

## الهدف من هذا الدرس

إبقاء قيم المستخدم خارج نص SQL، في `pg` و Prisma و TypeORM و Drizzle.

## المشكلة التي يحلها

بحث يبني الاستعلام هكذا:

```ts
`SELECT id, title FROM "Post" WHERE title ILIKE '%${q}%'`
```

إن كان `q` هو `' OR 1=1 --` يتغير معنى الاستعلام. قد تُعاد صفوف لا تخص البحث، أو يُنفَّذ أمر إضافي إن سمحت الصلاحية. التحقق من أن `q` سلسلة لا يمنع ذلك: النص ما زال SQL.

## الفكرة ببساطة

محرك القاعدة يفصل النص عن القيم. العنصر النائب `$1` في PostgreSQL، والقيمة في مصفوفة منفصلة. المحرك لا يفسّر القيمة كأوامر. أي مسار يلصق النص يعيد الخطر، حتى داخل ORM.

## التعريف التقني

`node-postgres` مع موضع:

```ts
await pool.query(
  `SELECT id, title FROM "Post" WHERE title ILIKE $1 LIMIT $2`,
  [`%${q}%`, 20],
);
```

علامة `%` هنا حرف بحث `LIKE` داخل القيمة، لا لصق SQL. `q` نفسه لا يُقحم في النص.

Prisma: الاستعلام الموسوم يمرّر القيم معاملات.

```ts
const rows = await prisma.$queryRaw<Array<{ id: number; title: string }>>`
  SELECT id, title FROM "Post" WHERE title ILIKE ${"%" + q + "%"} LIMIT ${20}
`;
```

`$queryRawUnsafe` يأخذ نصًا واحدًا. لصق `q` داخله يعيد الحقن. استخدمه فقط لنص ثابت كتبته أنت.

TypeORM: `dataSource.query(sql, [param])` مع مواضع. بناء النص بقالب من `req.query` هو الخطأ. `synchronize` لا علاقة له بالحقن.

Drizzle: قالب `sql` يربط القيم. `sql.raw(q)` يدرج النص كما هو. لا تمرّر بحث المستخدم إلى `raw`.

`ValidationPipe` و `@IsString` يفحصان الشكل قبل المعالج. لا يحوّلان الاستعلام إلى معاملات. أنبوب نظيف ثم لصق في SQL يبقى حقنًا.

Prisma `findMany({ where: { title: { contains: q } } })` يبني الاستعلام بمعاملات. فضّله على SQL خام للبحث العادي.

## Mental Model

```text
نص الاستعلام ثابت
القيمة في قناة ثانية
المحرك يقارن النص لا ينفّذه
لصق القالب يلغي القناة الثانية
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service.js";

@Injectable()
export class SearchService {
  constructor(private readonly prisma: PrismaService) {}

  titles(q: string) {
    return this.prisma.post.findMany({
      where: { title: { contains: q, mode: "insensitive" } },
      take: 20,
      select: { id: true, title: true },
    });
  }
}
```

## شرح المثال

`contains` يولّد `LIKE` مع معامل. `q` فيه اقتباس أو `--` يُبحث عنه كحروف في العنوان. `take: 20` حد التطبيق، منفصل عن الأمان، حتى لا يعيد البحث الجدول كله. `select` لا يسحب `passwordHash` من جدول آخر بالخطأ.

إن احتجت ترتيبًا لا يعبر عنه العميل، أبقِ `ORDER BY` في النص الثابت. لا تضع `order` القادم من الاستعلام داخل القالب.

## ماذا يحدث داخليًا؟

البروتوكول يرسل الاستعلام ثم القيم. المخطط يربط `$1` بعد التحليل. لذلك `' OR 1=1` يصبح نمط بحث لا شرطًا منطقيًا. في اللصق، المحلل يرى الشرط الجديد قبل التنفيذ.

الصلاحيات على دور القاعدة ما زالت تهم: دور التطبيق لا يملك `DROP`. ذلك يضيّق الضرر ولا يجعل اللصق آمنًا.

فهارس `title` بحث لاحق في فصل القوائم. الحقن مستقل عن وجود الفهرس.

## مثال عملي

اختبار يمرّر `q = "' OR 1=1 --"` إلى `titles` ويتوقع ألا تعود كل المقالات، بل ما يطابق النص حرفيًا، غالبًا مصفوفة فارغة. الاختبار نفسه على دالة تلصق القالب يفشل لأنه يعيد الصفوف. أضف الاختبار على أي `$queryRawUnsafe` بقي في المشروع.

مراجعة: ابحث عن `queryRawUnsafe` و `sql.raw` و `.query(\`` . كل تطابق مع إدخال مستخدم يُعاد كتابته بمعاملات.

## أخطاء شائعة

- تهريب يدوي للاقتباس بدل معامل.
- `ORDER BY ${sort}` لأن Prisma لا يرتب بنص حر في `findMany` فينتقل أحدهم إلى SQL خام.
- ظن أن ORM يمنع الحقن في كل الدوال. الدالة غير الآمنة تكسر ذلك.
- وضع المنطق الأمني في `WHERE` الملصوق ثم الاكتفاء بـ `forbidNonWhitelisted`.

## متى نستخدمه؟

معاملات أو دوال ORM للمعاملات في كل استعلام يلمس إدخالًا خارجيًا.

## متى لا نستخدمه؟

كبديل عن تفويض الصف. استعلام صحيح قد يعيد مقال Ada إن نسيت `authorId` أو `published`. الحقن والتخويل سؤالان.

## مقارنة

| معامل / `findMany` | لصق نص |
| --- | --- |
| القيمة بيانات | القيمة قد تصبح SQL |
| `$queryRaw` موسوم | `$queryRawUnsafe` مع إدخال |
| `sql` في Drizzle | `sql.raw` مع إدخال |

## تمارين

### تمرين سهل

هل `@IsString()` على `q` يمنع الحقن؟

<details>
<summary>الحل</summary>

لا. السلسلة قد تكون SQL. الفصل يتم بالمعامل أو بدالة ORM التي تربط القيمة.

</details>

### تمرين متوسط

اكتب استدعاء `pool.query` يبحث بعنوان مع حد `20`.

<details>
<summary>الحل</summary>

`pool.query('SELECT id, title FROM "Post" WHERE title ILIKE $1 LIMIT $2', ['%' + q + '%', 20])`.

</details>

### تمرين متقدم

لماذا يبقى `ORDER BY ${sort}` خطيرًا حتى لو كان `sort` واحدًا من قائمتك في أغلب الطلبات؟

<details>
<summary>الحل</summary>

طلب واحد خارج القائمة يغيّر الاستعلام إن لُصق. اقبل المفتاح في `@IsIn` ثم اختر عمودًا من خريطة ثابتة في الكود، ولا تدرج النص في SQL.

</details>

## أسئلة مراجعة

- ما فرق `$queryRaw` عن `$queryRawUnsafe`؟
- أين توضع `%` في بحث `LIKE`؟
- هل الفهرس يمنع الحقن؟

## الخلاصة

الاستعلام الثابت والقيمة المنفصلة. ORM الآمن يربط القيم. أي دالة تستقبل SQL نصيًا تحتاج أن يكون النص منك وحدك.

## ماذا بعد؟

[Validation كطبقة أمن](./07-validation-as-security.md) ماذا يوقف الأنبوب وماذا يتركه.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- NestJS — TypeORM: https://docs.nestjs.com/data/typeorm
- NestJS — Drizzle: https://docs.nestjs.com/data/drizzle
