# Prisma: CRUD

## الهدف من هذا الدرس

تنفيذ إنشاء كتاب وقراءته وتحديثه وحذفه من خدمة تحقن `PrismaService`، لا من المتحكم.

## المشكلة التي يحلها

استدعاء `prisma.book.create` داخل المسار يربط العقد بعميل القاعدة ويمنع استبدال العميل في اختبار وحدة. تمرير `@Body()` كاملًا إلى `data` يكتب أي حقل يرسله العميل إن طابق النوع، بما فيه حقول لم يقصدها العقد.

## الفكرة ببساطة

`BooksService` دوال صريحة: `create` و `findOne` و `list` و `update` و `remove`. كل دالة تستقبل حقولًا سمّيتها. المتحكم، حين يظهر في فصل REST، يمرّر DTO لا الكائن الخام. هنا نثبت شكل الاستدعاء على Prisma.

## التعريف التقني

صفحة Nest تعرض العمليات على النموذج: `findUnique` و `findMany` و `create` و `update` و `delete`. المعاملات أنواع مولَّدة مثل `Prisma.BookWhereUniqueInput` و `Prisma.BookCreateInput`.

```ts
return this.prisma.book.findUnique({ where: { id } });
```

`findUnique` يرجع `null` إن غاب الصف. الخدمة أو المتحكم يحوّل `null` إلى `NotFoundException` حين يكون الغياب خطأ عقد، لا قيمة مقبولة.

`findMany` يقبل `where` و `orderBy` و `skip` و `take` و `cursor`. استخدمها للصفحة الأولى من القائمة. فصل لاحق يجعل الترقيم عقدًا قابلًا لإعادة الاستخدام.

`create({ data })` يدرج صفًا. `update({ where, data })` يعدّل صفًا موجودًا ويرمي إن غاب. `delete({ where })` يحذف صفًا موجودًا.

القيد `@unique` على البريد يظهر كخطأ Prisma عند التكرار. حوّله إلى `ConflictException` عند حدود HTTP، لا تسرّب رمز المشغّل إلى العميل.

## Mental Model

```text
BooksService.create({ title })
        |
 prisma.book.create({ data: { title } })
        |
 INSERT ... RETURNING
        |
 كائن Book
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service.js";

@Injectable()
export class BooksService {
  constructor(private readonly prisma: PrismaService) {}

  create(title: string) {
    return this.prisma.book.create({
      data: { title, published: false },
    });
  }

  findById(id: number) {
    return this.prisma.book.findUnique({ where: { id } });
  }
}
```

## شرح المثال

الخدمة تفرض `published: false` عند الإنشاء. العميل لا يختار حالة النشر في هذا المسار حتى لو أرسل الحقل، لأن التوقيع يستقبل `title` فقط. هذا حد أوضح من نشر `body` كاملًا.

`findById` قد يرجع `null`. لا ترمه `404` داخل طبقة تُستخدم أيضًا من مهمة تقول «إن وُجد». اجعل دالة HTTP هي التي ترمي، أو دالة `requireById` منفصلة.

## ماذا يحدث داخليًا؟

العميل يبني SQL بمعاملات ويرسله عبر `PrismaPg`. `create` يعيد الصف بعد `RETURNING`. الأنواع تُحذف وقت التشغيل، والفحص الحقيقي للعمود يتم في PostgreSQL: عنوان أطول من قيد أضفته في SQL يفشل حتى لو قبله TypeScript.

`update` بلا صف يطابق `where` يرفض الوعد بخطأ «record not found» من Prisma. امسكه إن كان العقد `404`.

## مثال عملي

نشر الكتاب دالة `publish(id)` تستدعي `update` وتضع `published: true`، ولا تستقبل كائنًا جزئيًا من العميل. قائمة العناوين المنشورة:

```ts
return this.prisma.book.findMany({
  where: { published: true },
  orderBy: { id: "asc" },
  take: 20,
});
```

`take: 20` حد مؤقت حتى فصل الترقيم. اختبار الوحدة يستبدل `PrismaService` بكائن `{ book: { create: vi.fn(), findUnique: vi.fn() } }` كما في صفحة Nest، فلا يُفتح اتصال.

## أخطاء شائعة

- `create({ data: body })` والجسم من نوع `any`.
- اعتبار `null` من `findUnique` استثناءً تلقائيًا.
- `findMany` بلا `take` على جدول سيكبر.
- التقاط كل الأخطاء وإرجاع `500` عام فيختفي تعارض البريد.

## متى نستخدمه؟

لكل كتابة وقراءة صف تتكرر في ميزة الكتب أو الأعضاء.

## متى لا نستخدمه؟

لاستعلام يعبر جداول كثيرة بشرط معقّد مرة واحدة. يمكن إبقاؤه SQL مربوطًا داخل الخدمة نفسها إن كان أوضح، بعد أن تثبت المعاملات.

## مقارنة

| في المتحكم | في `BooksService` |
| --- | --- |
| HTTP و Prisma معًا | Prisma خلف دالة |
| اختباره يحتاج طلبًا | اختباره يستبدل الخدمة أو العميل |
| الجسم قد يتسرب إلى `data` | الحقول مسمّاة |

## تمارين

### تمرين سهل

ماذا ترجع `findUnique` إن لم يوجد الكتاب `4`؟

<details>
<summary>الحل</summary>

`null`. تحويل ذلك إلى `404` قرار طبقة أعلى أو دالة `require`.

</details>

### تمرين متوسط

لماذا ترفض الخدمة `data: body` حتى لو كان الجسم فيه `title`؟

<details>
<summary>الحل</summary>

الجسم قد يحمل `published` أو `id` أو حقلًا زائدًا. التوقيع الضيق يكتب الحقول التي سمّتها القاعدة فقط.

</details>

### تمرين متقدم

اختبار وحدة لا يجوز أن يتصل بـ PostgreSQL. كيف تسجّل `PrismaService` في `TestingModule`؟

<details>
<summary>الحل</summary>

`{ provide: PrismaService, useValue: { book: { findUnique: vi.fn() } } }` بجانب `BooksService`. الخطاف `onModuleInit` لا يعمل لأن النسخة الحقيقية غير مُنشأة.

</details>

## أسئلة مراجعة

- أي دالة تدرج صفًا، وأي دالة ترجع `null` عند الغياب؟
- لماذا لا يُنشر جسم الطلب كما هو؟
- ماذا يحد `take` هنا؟

## الخلاصة

CRUD يعيش في الخدمة عبر دوال العميل المولَّد، بحقول صريحة، و `null` يُعالَج بوعي، والاختبار يستبدل `PrismaService`.

## ماذا بعد؟

[Prisma: Relations](./08-prisma-relations.md) يربط الكتاب بالعضو الذي استعاره.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
