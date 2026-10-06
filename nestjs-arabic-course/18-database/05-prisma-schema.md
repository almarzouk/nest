# Prisma: Schema

## الهدف من هذا الدرس

كتابة نماذج الكتب والأعضاء في `schema.prisma` بحيث يولَّد منها عميل ذو أنواع، من غير خلط عنوان الاتصال بالمخطط.

## المشكلة التي يحلها

أنواع TypeScript المكتوبة يدويًا تنحرف عن الجدول. حقل `published` في الكود وعمود `is_published` في القاعدة يعيشان حياتين. فريق ثانٍ لا يعرف أي الأعمدة إلزامية.

## الفكرة ببساطة

ملف `schema.prisma` يصف المولّد ومزوّد القاعدة والنماذج. أمر التوليد يبني `PrismaClient` ودوالًا تطابق النماذج. عنوان الاتصال ليس داخل كتلة `datasource` في Prisma 7 كما تعرضه صفحة Nest: المزوّد يُذكر في المخطط، والعنوان في `DATABASE_URL` عبر ملف تهيئة Prisma.

## التعريف التقني

تهيئة البداية لـ PostgreSQL:

```bash
npx prisma init --datasource-provider postgresql --output ../src/generated/prisma
```

المولّد في مشروع ESM:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}
```

```prisma
datasource db {
  provider = "postgresql"
}
```

`output` نسبة إلى ملف المخطط يضع العميل داخل `src` ليُترجم مع التطبيق. من غير الخيار قد يخرج المجلد خارج `src`. العميل المولَّد يُستثنى من Git، لذلك `npx prisma generate` يعمل في CI قبل البناء.

Prisma يستنتج صيغة الوحدات من `tsconfig.json`. مشروع Nest الحديث ESM يحصل على عميل ESM. استيراد نسبي بلاحقة `.js`.

حقول النموذج تقابل أعمدة. `@id` مفتاح أساسي. `@default(autoincrement())` يولّد العدد في PostgreSQL عبر التسلسل الذي ينشئه الترحيل. `@unique` قيد فرادة. `String?` عمود قابل لـ NULL. العلاقة حقل يشير إلى نموذج آخر، وتُفصَّل في درس العلاقات.

أنواع العميل مثل `Prisma.BookCreateInput` توجد وقت الترجمة. `ValidationPipe` لا يراها وقت التشغيل، لذلك جسم HTTP يبقى DTO.

## Mental Model

```text
schema.prisma
  Book { id, title, published }
  Member { id, email }
        |
 npx prisma generate
        |
 PrismaClient.book / .member
```

## أبسط مثال

```prisma
model Book {
  id        Int     @id @default(autoincrement())
  title     String
  published Boolean @default(false)
}

model Member {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String?
}
```

## شرح المثال

`title` إلزامي في القاعدة وفي نوع الإنشاء. `name` اختياري. `published` يبدأ `false` إن لم يُرسل. `email` لا يتكرر. هذه النماذج بلا علاقة بعد، حتى يركز هذا الدرس على شكل الصف.

لا تضع كلمة مرور العضو في هذا النموذج ضمن الحقول التي تُرجَع للعميل لاحقًا. إن أضفت `passwordHash` فالاستجابة تحتاج إخفاءه في فصل التسلسل.

## ماذا يحدث داخليًا؟

`prisma generate` يقرأ المخطط ويكتب عميلًا فيه نوع لكل نموذج ودوال `findMany` و `create` وغيرها. تغيير المخطط من غير توليد يترك TypeScript على الشكل القديم، فتكتب حقلًا لا يعرفه العميل أو العكس.

الملف لا يحدّث PostgreSQL وحده. تطبيق الشكل على القاعدة هو الترحيل. يمكن أن يتقدم المخطط على القاعدة إن نسيت الأمر، فيفشل أول استعلام لأن العمود غير موجود.

## مثال عملي

مكتبة الحي تبدأ بهذين النموذجين. خدمة الكتالوج ستستدعي `prisma.book` بعد التوليد. اختبار الترجمة يفشل إن كتبت `prisma.book.create({ data: { titel: "x" } })` لأن الحقل ليس في المخطط. هذا أرخص من اكتشاف العمود الناقص على الخادم، بشرط أن يكون التوليد قد تم بعد آخر تعديل.

أضف `.env` والمجلد المولَّد إلى `.gitignore` كما يفعل `prisma init`.

## أخطاء شائعة

- تعديل المخطط ونسيان `prisma generate`.
- وضع `DATABASE_URL` السري داخل `schema.prisma` المرفوع.
- اعتبار نوع Prisma بديلًا عن DTO على `@Body`.
- نموذج بعشرين حقلًا تُعاد كلها في JSON.

## متى نستخدمه؟

لكل جدول ستقرأه الدورة عبر Prisma: كتب، أعضاء، ثم مستخدمون ومنشورات حين يبنى مشروع REST.

## متى لا نستخدمه؟

لوصف جسم HTTP أو شكل الاستجابة. تلك أصناف DTO ولاحقًا تسلسل.

## مقارنة

| `schema.prisma` | DTO |
| --- | --- |
| شكل الجدول والعميل | شكل طلب HTTP |
| يولّد أنواع استعلام | يفحصه `ValidationPipe` |
| يتغير مع الترحيل | يتغير مع العقد |

## تمارين

### تمرين سهل

أي حقل في `Member` يقبل الغياب، وأي حقل يرفض التكرار؟

<details>
<summary>الحل</summary>

`name` اختياري بسبب `String?`. `email` فريد بسبب `@unique`.

</details>

### تمرين متوسط

لماذا يُشغَّل `prisma generate` في CI؟

<details>
<summary>الحل</summary>

المجلد المولَّد غير مرفوع. البناء يحتاج العميل وأنواعه قبل `tsc`.

</details>

### تمرين متقدم

الحقل موجود في `schema.prisma` وبعد التوليد، والاستعلام يفشل لأن العمود غير موجود في PostgreSQL. ماذا لم يُشغَّل؟

<details>
<summary>الحل</summary>

الترحيل الذي ينفّذ SQL على القاعدة. التوليد يحدّث العميل فقط.

</details>

## أسئلة مراجعة

- أين يُذكر `postgresql`، وأين يُذكر العنوان؟
- ماذا ينتج `prisma generate`؟
- لماذا لا يصلح `BookCreateInput` جسمًا يفحصه الأنبوب وحده؟

## الخلاصة

المخطط مصدر شكل الجداول للعميل. التوليد يحدّث TypeScript. القاعدة نفسها تنتظر ترحيلًا يطبّق SQL.

## ماذا بعد؟

[Prisma: Migrations](./06-prisma-migrations.md) يحفظ SQL الذي ينشئ هذه الجداول.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- Prisma — Schema: https://www.prisma.io/docs/orm/prisma-schema
