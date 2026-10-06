# Status Codes

## الهدف من هذا الدرس

تثبيت رموز حالة كتالوج رف بحيث يفهمها العميل من الرقم: إنشاء 201، قائمة فارغة 200، غياب فرد 404، وتعارض حالة 409.

## المشكلة التي يحلها

`GET /books` من غير نتائج يعيد 404 فتعرض الواجهة «المسار غير موجود». `POST /books` يعيد 200 فلا تميّز الواجهة الإنشاء من القراءة. `DELETE` يعيد جسم الكتاب المحذوف مع 200 في مكان و 204 في مكان آخر.

## الفكرة ببساطة

الرمز قرار مختصر قبل قراءة JSON. النجاح الذي يعيد تمثيلًا يستخدم 200. الإنشاء يستخدم 201. الغياب لفرد محدد 404. المجموعة التي لا تحتوي عناصر ما زالت المجموعة، فترد 200 ومصفوفة فارغة.

## التعريف التقني

في Nest، إعادة كائن من الـ Handler تكتب 200 ضمن Standard response handling. `@HttpCode(201)` أو `@HttpCode(HttpStatus.NO_CONTENT)` يغيّر النجاح. الاستثناءات تحمل الرمز: `NotFoundException` 404، `ConflictException` 409، `BadRequestException` 400. استراتيجية الأخطاء في الفصل السابق تترجم أصناف المجال إلى هذه الرموز؛ هذا الدرس يثبت أي رمز يريده عقد رف. 201 للإنشاء حتى لو بنى المعرّف العميل في حالات نادرة؛ في رف الخادم يملك `id` ويرده في الجسم مع 201. 204 للحذف عندما لا يوجد جسم. لا تستخدم 404 لقائمة فارغة ولا 200 لإنشاء.

## Mental Model

```text
GET  /books        200  []
GET  /books/missing 404
POST /books        201  الجسم الجديد
PATCH موجود        200
DELETE بلا جسم     204
تعارض نشر          409
```

## أبسط مثال

```ts
import { Controller, Get, Post, Body, HttpCode, Param, NotFoundException } from '@nestjs/common';

@Controller('books')
export class BooksStatusController {
  private readonly rows = [{ id: 'b1', title: 'موسم الهجرة' }];

  @Get()
  list() {
    return this.rows.filter((book) => book.title.includes('لا شيء'));
  }

  @Post()
  @HttpCode(201)
  create(@Body() body: { title: string }) {
    const book = { id: 'b2', title: body.title };
    this.rows.push(book);
    return book;
  }

  @Get(':id')
  find(@Param('id') id: string) {
    const book = this.rows.find((row) => row.id === id);
    if (!book) throw new NotFoundException('Book not found');
    return book;
  }
}
```

## شرح المثال

- `list` تعيد `[]` والوضع الافتراضي 200. التصفية التي لا تطابق ليست غياب مسار.
- `@HttpCode(201)` لازم لأن الافتراضي 200 حتى على `POST`.
- `find` يرمي 404 للفرد فقط.

## ماذا يحدث داخليًا؟

1. المعالج يعيد قيمة أو يرمي `HttpException`.
2. إن أعاد قيمة، Nest يسلسلها ويضع 200 ما لم يوجد `@HttpCode` أو `res.status` صريح.
3. 201 تخبر العميل أن الفرد وُجد بسبب هذا الطلب، والجسم هو التمثيل.
4. 204 لا جسم له. لا ترجع كائنًا مع `@HttpCode(204)` ثم تتوقع أن يقرأه العميل.
5. 409 يأتي من تعارض حالة كما في استراتيجية الأخطاء، لا من قائمة فارغة.

## مثال عملي

```ts
import { Delete, HttpCode, Param } from '@nestjs/common';

@Delete(':id')
@HttpCode(204)
remove(@Param('id') id: string) {
  const index = this.rows.findIndex((row) => row.id === id);
  if (index < 0) throw new NotFoundException('Book not found');
  this.rows.splice(index, 1);
}
```

الحذف الناجح بلا جسم. الحذف لمعرّف غير موجود 404 لأن الفرد كان مطلوبًا. هذا يختلف عن `GET /books?author=لا-أحد` الذي يعيد 200 و `[]`.

## أخطاء شائعة

- 404 للقائمة الفارغة.
- 200 على `POST` الذي أنشأ صفًا.
- 500 لسعر مرفوض بينما العميل يستطيع التصحيح (400) أو للحالة الممنوعة (409).
- 201 مع جسم فارغ فلا يعرف العميل `id`.

## متى نستخدمه؟

في كل Handler جديد، قبل كتابة رسالة الخطأ. الرمز أولًا.

## متى لا نستخدمه؟

لا تخترع رمزًا غير مستخدم في العقد «ليدل على نوع الكتاب». المعنى التجاري في الجسم. الرمز من مجموعة صغيرة ثابتة.

## مقارنة

| الحالة | الرمز |
| --- | --- |
| قائمة، وفيها صفر كتب | 200 و `[]` |
| `GET` بمعرّف غائب | 404 |
| `POST` أنشأ كتابًا | 201 والجسم |
| حذف بلا جسم | 204 |
| ISBN مكرر | 409 |
| شكل JSON مرفوض | 400 |

## تمارين

### تمرين سهل

`GET /books?q=عنوان-غير-موجود` أعاد 404. ما الرمز الصحيح وما الجسم؟

<details>
<summary>الحل</summary>

200 و `[]`. 404 تُحفظ لـ `GET /books/:id` عندما لا يوجد ذلك الفرد.

</details>

### تمرين متوسط

`POST /books` يعيد الكتاب مع `id` لكن الوضع 200. ماذا يضيف المتحكم؟

<details>
<summary>الحل</summary>

`@HttpCode(201)` أو `HttpStatus.CREATED`. الجسم وحده لا يعلن الإنشاء. الافتراضي في Nest للرد العادي 200.

</details>

### تمرين متقدم

`DELETE /books/b1` والكتاب موجود في طلب شراء مفتوح. أي رمز، ولماذا ليس 404؟

<details>
<summary>الحل</summary>

409. الكتاب موجود لكن الحالة تمنع الحذف. 404 تقول إنه غير موجود فيكذب على العميل الذي يرى الكتاب في القائمة.

</details>

## أسئلة مراجعة

- ما رمز القائمة الفارغة؟
- لماذا `POST` الناجح في رف ليس 200؟
- متى 204 ومتى 200 بعد التعديل؟
- أي رمز لتعارض ISBN؟

## الخلاصة

200 للقراءة بما فيها المصفوفة الفارغة، 201 لإنشاء يعيد التمثيل، 204 لحذف بلا جسم، 404 لفرد غائب، 409 لتعارض حالة. `@HttpCode` يغيّر افتراض Nest.

## ماذا بعد؟

الإنشاء قد يصل مرتين بسبب إعادة محاولة على شبكة الجوال. الدرس التالي يجعل التكرار آمنًا بمفتاح Idempotency.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- MDN — 201 Created: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/201
- MDN — 404 Not Found: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/404
