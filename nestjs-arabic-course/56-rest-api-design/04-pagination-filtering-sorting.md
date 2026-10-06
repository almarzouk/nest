# Pagination و Filtering و Sorting كعقد

## الهدف من هذا الدرس

تثبيت شكل قائمة كتب رف: أي معاملات تُقبل، ماذا تعني الصفحة، كيف يُرتَّب، وماذا يعود عندما لا توجد نتائج — 200 ومصفوفة فارغة لا 404.

## المشكلة التي يحلها

عميل يرسل `page=1` وآخر يرسل `offset=0` وثالث `sort=published` بلا اتجاه. كل خدمة تفسر الغياب بشكل. واجهة الرف تحسب «الصفحة التالية» من طول المصفوفة فتتوقف مبكرًا أو تدور. التصفية `author` أحيانًا تُطابق جزئيًا وأحيانًا حرفيًا.

## الفكرة ببساطة

القائمة عقد مكتوب: أسماء المعاملات، البداية، الحد الأقصى، الترتيب الافتراضي، وشكل الغلاف حول المصفوفة. التصفية مساواة على حقول معلنة لا SQL حر. الترتيب حقل واتجاه من قائمة مغلقة. صفر نتائج نجاح.

## التعريف التقني

معاملات الاستعلام عبر `@Query()`. تحقق منها بـ Pipe حتى يُرفض `limit=10000` بـ 400. في رف: `limit` افتراضي 20 وأقصى 50، و `cursor` معرّف آخر كتاب في الصفحة السابقة (أنسب من `page` العددي عندما تُدرج كتب جديدة أثناء التصفح). `sort` قيم مغلقة مثل `publishedAt` و `direction` إما `asc` أو `desc`. الغلاف:

```json
{ "data": [], "nextCursor": null }
```

`data` فارغة مع 200 عندما لا تطابق التصفية. 404 لا تُستخدم هنا. `nextCursor: null` يعني لا صفحة تالية، حتى لو كان `data.length === limit` في حالات الحافة؛ لا تستنتج النهاية من الطول وحده إن كان العقد يعد بالمؤشر.

## Mental Model

```text
GET /books?author=نجيب&limit=20&cursor=b20&sort=publishedAt&direction=desc
→ 200 { data, nextCursor }
لا صفوف → 200 { data: [], nextCursor: null }
```

## أبسط مثال

```ts
import { Controller, Get, Query } from '@nestjs/common';

@Controller('books')
export class BooksListController {
  @Get()
  list(@Query('limit') limit = '20') {
    const take = Math.min(Number(limit) || 20, 50);
    return { data: [], nextCursor: null, limit: take };
  }
}
```

## شرح المثال

- الغلاف كائن لا مصفوفة عارية، حتى تستطيع إضافة `nextCursor` بلا كسر لاحقًا إن بدأتم به من أول العقد.
- القيمة العارية `[]` أيضًا 200، لكن إضافة مؤشر لاحقًا تكسر العميل. ابدأ بالغلاف.
- `Math.min` يحمي القاعدة؛ الرفض بـ 400 أوضح من القص الصامت إن كان العقد يقول «فوق 50 خطأ».

## ماذا يحدث داخليًا؟

1. الـ Pipe يتحقق من الأرقام والقيم المغلقة.
2. الخدمة تبني `where` من حقول مسموحة فقط: `author` مساواة، لا سلسلة `filter` يلصقها العميل.
3. `orderBy` من `sort` المعلن. فهرس الفصل السابق يطابق هذا الترتيب.
4. `take: limit + 1` حيلة شائعة لمعرفة وجود تالية، ثم تُقص الزيادة ويُملأ `nextCursor`. إن لم تزد، `nextCursor` يكون `null`.
5. النتيجة تُغلَّف. لا استثناء «فارغ».

## مثال عملي

```ts
import { BadRequestException, Injectable } from '@nestjs/common';

const sorts = new Set(['publishedAt', 'title']);

@Injectable()
export class BooksQuery {
  normalize(input: { limit?: string; sort?: string; direction?: string; author?: string }) {
    const limit = Number(input.limit ?? 20);
    if (!Number.isInteger(limit) || limit < 1 || limit > 50) {
      throw new BadRequestException('limit must be 1..50');
    }
    const sort = input.sort ?? 'publishedAt';
    if (!sorts.has(sort)) throw new BadRequestException('unsupported sort');
    const direction = input.direction ?? 'desc';
    if (direction !== 'asc' && direction !== 'desc') {
      throw new BadRequestException('unsupported direction');
    }
    return { limit, sort, direction, author: input.author };
  }
}
```

`author` غائب يعني بلا تصفية لا خطأ. `author` حاضر بلا نتائج يبقى 200. المؤشر التالف 400 لأنه طلب غير مفهوم، لا 404.

## أخطاء شائعة

- 404 عند `data: []`.
- `sort` يقبل أي اسم عمود فيصل العميل إلى حقل داخلي أو يطيح بالاستعلام.
- صفحة رقمية بلا ترتيب ثابت فيتكرر كتاب بين الصفحتين عند إدراج جديد.
- تغيير اسم `limit` إلى `pageSize` في نسخة صامتة. هذا كسر عقد، لا تحسين.

## متى نستخدمه؟

على كل مجموعة قد تكبر: `/books` و `/books/b1/reviews`. اكتب المعاملات في وصف OpenAPI مع المسار.

## متى لا نستخدمه؟

على `GET /books/:id`. الفرد ليس صفحة. ولا تمرر `order by` كنص خام.

## مقارنة

| النتيجة | الرمز | الجسم |
| --- | --- | --- |
| عشرون كتابًا وثمة مزيد | 200 | `data.length` حتى الحد و `nextCursor` |
| لا كتب للمؤلف | 200 | `data: []` و `nextCursor: null` |
| `limit=999` | 400 | لا قائمة |
| `sort=password` | 400 | الحقل غير معلن |

## تمارين

### تمرين سهل

مؤلف بلا كتب. ما الوضع وما `data`؟

<details>
<summary>الحل</summary>

200 و `data: []` و `nextCursor: null`. المسار موجود والتصفية صحيحة.

</details>

### تمرين متوسط

لماذا المؤشر أثبت من `page=3` في كتالوج تُنشر فيه كتب أثناء تصفح العميل؟

<details>
<summary>الحل</summary>

الصفحة العددية تنزلق عندما يُدرج صف في الترتيب قبلها، فيتكرر عنصر أو يختفي. المؤشر يقول «بعد هذا الكتاب في هذا الترتيب»، فيكمل من مكانه.

</details>

### تمرين متقدم

العميل يحسب وجود صفحة تالية من `data.length === limit`. متى يخطئ؟

<details>
<summary>الحل</summary>

عندما يكون عدد النتائج مطابقًا للحد تمامًا ولا توجد زيادة. سيطلب صفحة أخرى فارغة أو يظن أن ثمة مزيدًا. العقد الذي يعد بـ `nextCursor` يجعل `null` هو النهاية، والخدمة تعرف ذلك بطلب عنصر إضافي لا يُعاد في `data`.

</details>

## أسئلة مراجعة

- ما رمز القائمة الفارغة؟
- أي حقول يُسمح بترتيبها؟
- لماذا الغلاف أفضل من مصفوفة عارية من أول يوم؟
- متى 400 على معاملات القائمة؟

## الخلاصة

Pagination عقد: حد، ترتيب مغلق، تصفية معلنة، غلاف فيه `nextCursor`. لا نتائج تعني 200 و `data: []`. 404 ليست نتيجة بحث فاضية.

## ماذا بعد؟

الـ 400 والـ 404 والـ 409 تحتاج جسمًا واحدًا تتفق عليه الواجهة. الدرس التالي شكل الخطأ.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- MDN — 200 OK: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/200
