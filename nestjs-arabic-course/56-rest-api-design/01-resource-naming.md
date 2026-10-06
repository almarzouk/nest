# تسمية الموارد

## الهدف من هذا الدرس

تسمية عناوين متجر رف بأسماء أشياء لا بأسماء أفعال، حتى يُقرأ `GET /books/b1` قبل فتح الكود.

## المشكلة التي يحلها

المسارات نمت كدوال: `/createBook`، `/getBookById`، `/doPublish`. كل فعل أسلوب جديد، والعميل لا يستنتج الحذف من الإنشاء. التوثيق يعيد شرح كل عنوان كأنه مشروع مستقل.

## الفكرة ببساطة

المورد اسم مجموعة: `books`، `authors`، `orders`. المعرّف يحدد فردًا. فعل HTTP يقول ماذا نفعل به. العملية التي ليست شيئًا يُخزَّن تُسمّى كمورد فرعي أو كفعل آخر في المجموعة عندما تضيق الأفعال الخمسة، لا كجملة إنجليزية في المسار.

## التعريف التقني

المسار في `@Controller('books')` و `@Get(':id')` يبني `/books/:id`. الأسماء جمع، أحرف صغيرة، كلمات مفصولة بشرطة إن لزم: `/shelf-sections`. لا فعل في المسار طالما `POST` و `PATCH` و `DELETE` كافية. العلاقات التي يملكها الكتاب تُعلَّق تحته: `/books/b1/reviews`. المعرّف في المسار لا في الاستعلام للفرد الواحد. الاستعلام للتصفية على المجموعة، ودرسه لاحقًا.

## Mental Model

```text
مجموعة   /books
فرد      /books/b1
ملكية    /books/b1/reviews
فعل HTTP فوق الاسم، لا داخل الاسم
```

## أبسط مثال

```ts
import { Controller, Get, Param, Post, Body, Patch, Delete } from '@nestjs/common';

@Controller('books')
export class BooksController {
  @Get()
  list() {
    return [];
  }

  @Post()
  create(@Body() body: { title: string }) {
    return { id: 'b1', title: body.title };
  }

  @Get(':id')
  find(@Param('id') id: string) {
    return { id };
  }

  @Patch(':id')
  update(@Param('id') id: string) {
    return { id };
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return { id };
  }
}
```

## شرح المثال

- خمسة عناوين لا خمسة أفعال في النص.
- `create` اسم الدالة داخلي. العميل يرى `POST /books`.
- المعرّف `:id` فرد. القائمة بلا معرّف.

## ماذا يحدث داخليًا؟

1. Nest يدمج مسار المتحكم مع مسار الدالة.
2. `POST /books` ينشئ فردًا في المجموعة.
3. `PATCH /books/b1` يغير حقولًا في الفرد.
4. `POST /books/b1/publish` مقبول عندما «النشر» عملية لا تُختصر في حقل، ويُوثَّق كمورد عملية. إن كان النشر مجرد `status`، فـ `PATCH` على الكتاب أوضح.
5. الاسم الداخلي للجدول في Prisma لا يلزم أن يساوي المسار، لكن التباعد الكبير يربك الفريق. اكتب التعيين في مكان واحد.

## مثال عملي

مراجعات كتاب في رف، لا مسار `/addReview`:

```ts
@Controller('books/:bookId/reviews')
export class ReviewsController {
  @Get()
  list(@Param('bookId') bookId: string) {
    return [{ bookId, id: 'r1', body: 'واضح' }];
  }

  @Post()
  create(@Param('bookId') bookId: string, @Body() body: { body: string }) {
    return { id: 'r2', bookId, body: body.body };
  }
}
```

القراءة `GET /books/b1/reviews`. الإنشاء `POST` على المجموعة الفرعية. حذف مراجعة `DELETE /books/b1/reviews/r1`.

## أخطاء شائعة

- `/getBooks` و `/api/book` مفردًا مرة وجمعًا مرة.
- فعل في المسار مع `POST` فيتكرر المعنى: `POST /createBook`.
- تعليق معرّف الأب في الجسم فقط: `POST /reviews` مع `bookId` مخفي، بينما العنوان لا يقول لمن المراجعة.
- خلط لغة المسارات بين العربية والإنجليزية. المسارات في هذه الدورة إنجليزية ثابتة.

## متى نستخدمه؟

من أول متحكم في رف. أي قدرة جديدة تُسمّى مجموعة أو فردًا قبل كتابة الدالة.

## متى لا نستخدمه؟

لا تحوّل كل عملية إلى اسم مورد إن كان `PATCH` لحقل واحد يكفي ويبقى المفهوم أوضح. ولا تسمِّ مسارات داخلية للمهام بأسماء عامة «لأن REST».

## مقارنة

| المطلوب | مسار أوضح |
| --- | --- |
| كل الكتب | `GET /books` |
| كتاب | `GET /books/b1` |
| مراجعاته | `GET /books/b1/reviews` |
| إنشاء كتاب | `POST /books` |
| تغيير السعر | `PATCH /books/b1` |

## تمارين

### تمرين سهل

اقترح بديلًا لـ `POST /publishBook`.

<details>
<summary>الحل</summary>

`POST /books/b1/publish` إن كان النشر عملية ذات أثر واسع، أو `PATCH /books/b1` بجسم `{ "status": "published" }` إن كان انتقال حالة فقط. لا تُبقِ الفعل الإنجليزي بلا معرّف في جذر الـ API.

</details>

### تمرين متوسط

لماذا `/books/b1/reviews` أوضح من `/reviews?bookId=b1` لمراجعة يملكها كتاب؟

<details>
<summary>الحل</summary>

لأن العنوان يعلن الملكية: لا مراجعة في هذا المسار بلا كتاب. التصفية بالاستعلام تناسب البحث عبر الكتب، لا الكيان الابن الدائم.

</details>

### تمرين متقدم

رف تريد «إعارة» تربط كتابًا بعميل لمدة. هل المسار فعل أم مورد؟

<details>
<summary>الحل</summary>

مورد `loans`: `POST /loans` بجسم فيه `bookId` و `userId`، ثم `GET /loans/l1`. الإعارة شيء يُتتبّع (تاريخ، إعادة)، ليست دالة تختفي. `POST /lendBook` يخفي هذا الشيء.

</details>

## أسئلة مراجعة

- أين يعيش الفعل: في المسار أم في طريقة HTTP؟
- متى تضع المعرّف في المسار؟
- كيف تُسمّى مجموعة تابعة لكتاب؟
- ما عيب `/getBooks`؟

## الخلاصة

عنوان رف اسم مجموعة أو فرد أو ملكية. الطريقة تحمل الفعل. العملية التي تُتتبّع تصبح موردًا لا جملة في المسار.

## ماذا بعد؟

الأسماء ثابتة. الدرس التالي يربط كل اسم برمز حالة، ومنها أن القائمة الفارغة 200 وأن الإنشاء 201.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- MDN — HTTP request methods: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods
