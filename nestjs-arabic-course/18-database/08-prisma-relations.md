# Prisma: Relations

## الهدف من هذا الدرس

ربط `Book` بعضو المكتبة الذي يستعيره عبر مفتاح أجنبي في المخطط، وتحميل العلاقة في الاستعلام نفسه عند الحاجة.

## المشكلة التي يحلها

تخزين اسم العضو كنص داخل صف الكتاب يكرّر الاسم ويكسره حين يصحّح العضو بريده. حلقة `for` تستعلم عن العضو لكل كتاب تصنع عشرات الرحلات إلى القاعدة.

## الفكرة ببساطة

العلاقة في Prisma حقل على النموذجين: الكتاب يشير إلى `memberId`، والعضو لديه قائمة `books`. العمود الحقيقي هو المفتاح الأجنبي في جدول الكتب. `include` أو `select` يحمّل الطرف الآخر في الاستعلام الذي تطلبه، لا في كل قراءة تلقائيًا.

## التعريف التقني

الشكل الذي تستخدمه صفحة Nest بين `User` و `Post` ينطبق هنا على `Member` و `Book`:

```prisma
model Member {
  id    Int    @id @default(autoincrement())
  email String @unique
  books Book[]
}

model Book {
  id        Int     @id @default(autoincrement())
  title     String
  published Boolean @default(false)
  member    Member? @relation(fields: [memberId], references: [id])
  memberId  Int?
}
```

`Book[]` طرف المجموعة. `Member?` طرف اختياري: كتاب على الرف بلا مستعير. `fields` و `references` تعرّفان المفتاح الأجنبي.

بعد تعديل المخطط: ترحيل ثم `prisma generate`.

قراءة مع الطرف الآخر:

```ts
this.prisma.book.findUnique({
  where: { id },
  include: { member: true },
});
```

كتابة متداخلة تنشئ العضو مع الكتاب في معاملة تلقائية، كما تذكر صفحة Nest عن nested writes. للدورة، أنشئ العضو أولًا ثم مرّر `memberId` حتى يبقى كل أثر صريحًا، إلا حين يكون الإنشاءان قرارًا واحدًا.

`onDelete` في `@relation` يحدّد ماذا يحدث لكتب العضو عند حذف العضو. الافتراضي في المثال الرسمي لـ Nest على علاقة اختيارية قد يظهر `ON DELETE SET NULL` في SQL المولَّد. اختر السلوك عمدًا: منع الحذف، أو فصل الكتب، لا مفاجأة.

## Mental Model

```text
members.id  <- books.memberId
Book.member     علاقة للعميل
include.member  تحميل في هذا الاستعلام فقط
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service.js";

@Injectable()
export class LoansService {
  constructor(private readonly prisma: PrismaService) {}

  loan(bookId: number, memberId: number) {
    return this.prisma.book.update({
      where: { id: bookId },
      data: { memberId },
      include: { member: { select: { id: true, email: true } } },
    });
  }
}
```

## شرح المثال

`data` يغيّر المفتاح الأجنبي فقط. `select` على العضو يعيد المعرّف والبريد، لا كل الأعمدة إن أضفت لاحقًا `passwordHash`. الكتاب غير الموجود يجعل `update` يفشل. عضو غير موجود يكسر قيد المفتاح الأجنبي. حوّل الخطأين إلى `404` مفهوم عند حدود HTTP.

## ماذا يحدث داخليًا؟

Prisma يولّد `UPDATE` ثم استعلامًا أو ضمًا لتحميل `member` حسب ما يطلبه `include`. من غير `include`، الكائن العائد فيه `memberId` ولا فيه كائن العضو. هذا متعمَّد حتى لا تُجلب الشجرة في كل مرة.

N+1: `findMany` بلا `include` ثم `prisma.member.findUnique` داخل `for` بعدد الكتب. الحل `include` على القائمة، أو `where: { id: { in: ids } }` بعدد استعلامات ثابت.

## مثال عملي

رف المكتبة: `findMany` للكتب المنشورة مع `member` عبر `select` ضيق. تقرير «كتب بلا مستعير» يستخدم `where: { memberId: null }` من غير تحميل علاقة. حذف عضو ما زال عنده كتب يفشل إن ضبطت القيد على المنع، فتعرض الخدمة رسالة تطلب إعادة الكتب أولًا.

## أخطاء شائعة

- تكرار اسم العضو في عمود نصي على الكتاب.
- `include` لكل الحقول الحساسة.
- حلقة استعلام لكل صف.
- نسيان الترحيل بعد إضافة `memberId`.

## متى نستخدمه؟

حين لكيان مالك حقيقي في جدول آخر، مثل المستعير أو مؤلف لاحق.

## متى لا نستخدمه؟

لقيمة صغيرة لا تتغير ولا تُستعلم وحدها، مثل رمز لغة من حرفين يمكن أن يبقى عمودًا بسيطًا.

## مقارنة

| عمود نصي منسوخ | مفتاح أجنبي |
| --- | --- |
| ينحرف عند التعديل | يشير إلى صف واحد |
| بلا قيد وجود | PostgreSQL يرفض معرّفًا مجهولًا |
| لا `include` | تحميل عند الطلب |

## تمارين

### تمرين سهل

أي عمود في الجدول يمثّل العلاقة؟

<details>
<summary>الحل</summary>

`memberId` في `books`. الحقل `member` وجه العميل لتلك العلاقة.

</details>

### تمرين متوسط

لماذا `select` على البريد أفضل من `include: { member: true }` إن كان للعضو حقل سرّي؟

<details>
<summary>الحل</summary>

`include` الكامل يجلب كل الأعمدة القابلة للقراءة على النموذج، ومنها السر إن وُجد. `select` يسمّي ما يخرج نحو الخدمة.

</details>

### تمرين متقدم

عشرون كتابًا، ولكل واحد استعلام عضو. أعد الكتابة باستعلام قائمة واحد.

<details>
<summary>الحل</summary>

`findMany({ include: { member: { select: { id: true, email: true } } } })` أو ما يعادل ضمًا واحدًا. لا تستدعِ `findUnique` داخل الحلقة.

</details>

## أسئلة مراجعة

- ماذا يعني `Member?` على الكتاب؟
- متى تُحمَّل العلاقة؟
- ماذا يمنع المفتاح الأجنبي؟

## الخلاصة

العلاقة عمود مفتاح أجنبي في المخطط، وتحميل صريح في الاستعلام، وحقول ضيقة حتى لا تخرج بيانات العضو كلها.

## ماذا بعد؟

[Prisma: Transactions](./09-prisma-transactions.md) يجمع إعارة الكتاب وخصم النسخة في أثر واحد.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- Prisma — Relations: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations
