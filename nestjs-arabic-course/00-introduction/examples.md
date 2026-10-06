# أمثلة عملية

هذا الملف يجمع دروس المقدمة على سيناريو واحد صغير: **متجر كتب**. لا تبني مشروع Nest هنا. الهدف أن تنتقل بين HTTP و REST و Express و خريطة Nest من غير قفز.

عندما يظهر كود Nest فهو معاينة للخريطة في [الدرس 16](./16-nestjs-architecture-overview.md). الوسوم تُشرح في فصولها.

## المثال 1 — عقد القراءة

عميل يريد كتابًا.

```http
GET /books/2 HTTP/1.1
Host: localhost:3000
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "2",
  "title": "Refactoring",
  "price": 40
}
```

- Client هو مرسل `GET`. Server هو عملية المتجر.
- المورد هو الكتاب `2`. التمثيل JSON، لا صف قاعدة البيانات.
- `price` عدد. لو أرسلناه `"40"` لتغير عقد الجمع عند العميل.

غياب الكتاب:

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "statusCode": 404,
  "message": "Book not found"
}
```

الرمز يحمل القرار. الجسم يشرح.

## المثال 2 — العقد كله لهذه المرحلة

```text
GET    /books
POST   /books
GET    /books/:id
PATCH  /books/:id
DELETE /books/:id
GET    /books/:id/reviews
POST   /books/:id/reviews
```

إنشاء:

```json
{ "title": "Clean Code", "price": 30 }
```

الخادم يعيد `201` ويضيف `id`. العميل لا يخترع المعرّف في هذا العقد.

`PATCH` يرسل الحقول التي تتغير فقط:

```json
{ "price": 28 }
```

`PUT` لم نضعه في الجدول لأننا لا نريد إجبار العميل على إرسال الكتاب كاملًا في هذا المتجر. هذا قرار تصميم، ويُراجع إذا صار للكتاب حقول كثيرة يجب استبدالها معًا.

## المثال 3 — Express صغير للكتب

طبقة JSON، ثم قراءة، ثم رفض الناقص عند الإنشاء. البيانات في الذاكرة حتى يبقى المثال قابلًا للقراءة.

```ts
import express, { type Request, type Response } from 'express';

const app = express();

interface Book {
  id: string;
  title: string;
  price: number;
}

const books: Book[] = [{ id: '2', title: 'Refactoring', price: 40 }];

app.use(express.json());

app.get('/books/:id', (req: Request, res: Response) => {
  const book = books.find((item) => item.id === req.params.id);
  if (!book) {
    res.status(404).json({ statusCode: 404, message: 'Book not found' });
    return;
  }
  res.json(book);
});

app.post('/books', (req: Request, res: Response) => {
  const title: unknown = req.body?.title;
  const price: unknown = req.body?.price;
  if (typeof title !== 'string' || title.length === 0 || typeof price !== 'number') {
    res.status(400).json({ statusCode: 400, message: 'Invalid book' });
    return;
  }
  const book: Book = { id: String(Date.now()), title, price };
  books.push(book);
  res.status(201).json(book);
});

app.listen(3000);
```

اقرأ السلسلة لطلب `POST /books` بجسم صالح:

1. `express.json()` يملأ `req.body` ثم ينادي `next` الداخلي فيه.
2. طبقة `POST` تفحص النوع وقت التشغيل. TypeScript وحده لا يرى جسم الشبكة.
3. عند النجاح ترسل `201` والجسم، ثم تتوقف. لا تستدعِ `next()` بعد `res.json`، لأن الرد انتهى.

المقابل الذي ستكتبه لاحقًا بعد فصول Controllers و Pipes يشبه الفكرة التالية، لا تنسخه إلى مشروع اليوم:

```ts
@Post()
create(@Body() dto: CreateBookDto): Book {
  return this.booksService.create(dto);
}
```

التحقق سيكون على `CreateBookDto` لا داخل `if` في كل مسار. الدالة تُرجع القيمة، و Nest تكتب الرد. توثيق Controllers يقول إن رمز النجاح الافتراضي `200`، وإن `POST` افتراضيًا `201`. تغيير ذلك يتم لاحقًا بـ `@HttpCode`. فصل Controllers يشرح هذا مع `@Res` عندما تحتاج التحكم اليدوي.

## المثال 4 — تتبع Nest لطلب واحد

افترض أن المتجر كبر وصارت فيه محطات حقيقية. طلب `PATCH /books/2` بسعر جديد من مستخدم مسجّل:

```text
1  الطلب يدخل Express
2  Middleware العام يضع JSON في الجسم
3  Guard الهوية يقبل الطلب
4  Interceptor يبدأ مؤقتًا
5  Pipe يتحقق أن price عددًا موجبًا
6  BooksController.update يستقبل id والقيمة الجاهزة
7  BooksService تطبّق القاعدة: الكتاب موجود، والسعر مسموح
8  Interceptor بعد المتحكم يسجّل المدة
9  لا استثناء، فالفلتر لا يعمل
10 200 و JSON للكتاب بعد التعديل
```

لو كان السعر `"مجاني"` كنص:

- المحطة 5 ترفض.
- المتحكم والخدمة لا يعملان.
- العميل يرى خطأ تحقق، غالبًا ضمن فئة `400` عندما نستخدم مسار Nest الافتراضي للـ ValidationPipe.

لو كانت الخدمة ترمي "الكتاب غير موجود" بعد أن تجاوز الشكل الفحص:

- المتحكم لا يلتقط الخطأ.
- فلتر المسار إن وُجد يحوّله إلى `404`.
- إن لم يوجد، الفلتر الأعم يفعل ذلك.
- لا نكتب `404` يدويًا في ثلاثة ملفات.

## المثال 5 — أين أضع الكود؟

| المطلوب | المكان |
| --- | --- |
| مسار `PATCH /books/:id` | Controller |
| "السعر لا يقل عن صفر" كقاعدة عمل بعد أن وصل رقم | Service |
| "الجسم يجب أن يحتوي price كعدد" | DTO + Pipe |
| "فقط موظف يغيّر السعر" | Guard |
| تسجيل مدة الطلب | Interceptor |
| شكل JSON عند الغياب | Exception Filter |
| مشاركة خدمة الكتب مع وحدة الطلبات | Module يصدّر الخدمة، والوحدة الأخرى تستورده |
| استدعاء Express API نادرة لمكتبة مجتمع | حافة الإقلاع أو المحوّل، لا داخل BooksService |

إذا ترددت بين صفين، أبقِ HTTP والكائنات `req` و `res` في الصف الأول والصف الأخير، وأبقِ القاعدة في Service.

## ماذا بعد؟

حل تمارين الفصل من غير الرجوع إلى هذا الملف أولًا. ثم اقرأ التمهيد إلى TypeScript.
