# Pagination

## الهدف من هذا الدرس

تقسيم `GET /posts` إلى صفحات صغيرة بـ `page` و `pageSize`، كمقدمة على هذا المسار فقط.

## المشكلة التي يحلها

`findMany` بلا حد يقرأ كل المقالات كلما كبر الجدول، فيطول الطلب ويكبر JSON. العميل الذي يريد «أول عشرين» لا يملك عقدًا مشتركًا إن أعاد كل مطوّر اسم استعلام مختلفًا.

## الفكرة ببساطة

الاستعلام `?page=1&pageSize=20`. الصفحة تبدأ من `1`. الخدمة تحسب `skip = (page - 1) * pageSize` وتمرّر `take = pageSize`. الاستجابة تعيد العناصر وعدد الصفحة وحجمها، والعدد الكلي إن كان رخيصًا على هذا الجدول. الفصل `26` يعمّم العقد ويناقش متى يصبح `skip` مكلفًا. هنا نثبّت سلوكًا واحدًا لقائمة المنشورات.

## التعريف التقني

DTO الاستعلام:

```ts
import { Type } from "class-transformer";
import { IsInt, IsOptional, Max, Min } from "class-validator";

export class ListPostsQueryDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(50)
  pageSize = 20;
}
```

`@Type` يلزم لأن الاستعلام نص. `Max(50)` يمنع `pageSize=100000`. الأنبوب يحتاج `transform: true`.

```ts
@Get()
list(@Query() query: ListPostsQueryDto) {
  return this.posts.list(query.page, query.pageSize);
}
```

```ts
async list(page: number, pageSize: number) {
  const skip = (page - 1) * pageSize;
  const [items, total] = await this.prisma.$transaction([
    this.prisma.post.findMany({
      skip,
      take: pageSize,
      orderBy: { id: "asc" },
      select: { id: true, title: true, published: true, authorId: true },
    }),
    this.prisma.post.count(),
  ]);
  return { items, page, pageSize, total };
}
```

المعاملة هنا لقطة للقائمة والعدد معًا، لا كتابة. الترتيب الافتراضي `id` حتى لا تقفز الصفوف بين الصفحات بلا `orderBy`. التصفية والترتيب بمعاملات إضافية درسان لاحقان. لا تبنِ مؤشرًا `cursor` في هذا الفصل.

`GET /posts/:id` ما زال يجلب واحدًا. لا تمرّر `page` إليه.

## Mental Model

```text
page=2 pageSize=20
        |
 skip 20 take 20
        |
 { items, page, pageSize, total }
```

## أبسط مثال

طلب:

```http
GET /posts?page=2&pageSize=20 HTTP/1.1
```

استجابة مختصرة:

```json
{
  "items": [],
  "page": 2,
  "pageSize": 20,
  "total": 25
}
```

إن كان العدد `25`، الصفحة `2` فيها خمسة عناصر. صفحة `3` مصفوفة فارغة مع `total` نفسه، وهذا أفضل من `404` لأن الصفحة خارجة عن النطاق ليست مقالًا غائبًا.

## شرح المثال

العميل يحسب عدد الصفحات بـ `Math.ceil(total / pageSize)` إن احتاج شريط صفحات. الخادم لا يلزم أن يرسل كل الأرقام. `items` بلا `body` في القائمة حتى تبقى خفيفة، والتفاصيل في `GET /posts/:id`. هذا قرار عقد، لا قيد Prisma.

## ماذا يحدث داخليًا؟

`OFFSET` في SQL يتخطى الصفوف. على صفحة `1` التكلفة صغيرة. على صفحة بعيدة الخادم ما زال يمر على المتروك. لذلك الفصل `26` يشرح المؤشر والفهرس. مقدمة هذا المشروع تقبل `skip` لأن حجم التمرين صغير والسقف `50`.

`count()` استعلام ثانٍ. على جدول عملاق قد يُؤجَّل العدد. هنا الجدول تعليمي والإرجاع مفيد.

## مثال عملي

ازرع `25` مقالًا. اطلب الصفحة `1` وتوقع `20` معرّفًا تصاعديًا. اطلب الصفحة `2` وتوقع الخمسة التالية من غير تداخل. أرسل `pageSize=500` وتوقع `400` بسبب `@Max(50)`. أرسل `page=0` وتوقع `400` بسبب `@Min(1)`.

## أخطاء شائعة

- `page` يبدأ من `0` في الخادم ومن `1` في التوثيق.
- قائمة بلا `orderBy` فتتبدّل الحدود.
- `pageSize` بلا سقف.
- بناء مساعد عام لكل الموارد قبل أن يعمل هذا المسار. التعميم لاحقًا.

## متى نستخدمه؟

على `GET /posts` حين تتجاوز النتيجة ما تريد إرساله دفعة واحدة.

## متى لا نستخدمه؟

على `GET /posts/:id`. وعلى تعليقات هذا الفصل إن كان السقف الثابت `50` كافيًا للتمرين. وحين تحتاج ثباتًا تحت إدراجات كثيفة: ذلك مؤشر في الفصل `26`.

## مقارنة

| الكل دفعة واحدة | `page` و `pageSize` |
| --- | --- |
| بسيط على عشرة صفوف | ثابت على ألف |
| بلا عقد استعلام | أسماء محددة |
| بلا `skip` | `skip` مقبول هنا |

## تمارين

### تمرين سهل

ما `skip` للصفحة `3` والحجم `10`؟

<details>
<summary>الحل</summary>

`(3 - 1) * 10 = 20`.

</details>

### تمرين متوسط

لماذا تُرفض صفحة فارغة بـ `200` ومصفوفة فارغة لا بـ `404`؟

<details>
<summary>الحل</summary>

المسار صحيح والمورد الموجود هو القائمة. `404` للمقال ذي المعرّف الغائب.

</details>

### تمرين متقدم

لماذا لا ننسخ هذا الـ DTO إلى التعليقات والمستخدمين في اليوم نفسه؟

<details>
<summary>الحل</summary>

العقد ما زال تجربة على المنشورات. الفصل `26` يستخرج النمط بعد أن تثبت الأسماء والحجم الأقصى وأثر `skip` على الفهرس.

</details>

## أسئلة مراجعة

- من أين يأتي `page` نصًا؟
- لماذا `orderBy` لازم مع `skip`؟
- أين يُشرح قصور `OFFSET`؟

## الخلاصة

قائمة المنشورات صفحة بحجم محدود مع `skip` و `take` وعدد كلي. هذا عقد مسار واحد، لا مكتبة ترقيم بعد.

## ماذا بعد؟

[Filtering](./08-filtering.md) يضيّق الصفحة بشرط مثل `published`.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- NestJS — Validation: https://docs.nestjs.com/application/validation
- NestJS — Prisma: https://docs.nestjs.com/data/prisma
