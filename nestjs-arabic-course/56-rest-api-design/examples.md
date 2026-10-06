# أمثلة عملية

عقد كتب رف في صفحة واحدة. الأرقام هنا هي العقد، والآلية في الفصول السابقة.

## المثال 1 — مورد وقائمة فارغة

```http
GET /v1/books?author=لا-أحد&limit=20 HTTP/1.1
```

```http
HTTP/1.1 200 OK

{ "data": [], "nextCursor": null }
```

المسار اسم مجموعة. التصفية بلا نتائج ليست 404.

## المثال 2 — إنشاء بمفتاح

```http
POST /v1/books HTTP/1.1
Idempotency-Key: 6f1c0c2e-7b1a-4d0a-9c2e-1a2b3c4d5e6f
Content-Type: application/json

{ "title": "موسم الهجرة إلى الشمال" }
```

```http
HTTP/1.1 201 Created

{ "id": "b1", "title": "موسم الهجرة إلى الشمال", "price": 2500 }
```

الإعادة بنفس المفتاح ونفس المعنى تعيد 201 و `b1`. جسم بعنوان آخر يعيد:

```json
{ "statusCode": 409, "code": "idempotency_conflict", "message": "Key reused with a different payload" }
```

## المثال 3 — نفس الكتاب في v2 ونفس الخطأ

```json
{ "id": "b1", "title": "موسم الهجرة إلى الشمال", "price": { "amount": 2500, "currency": "SAR" } }
```

`GET /v1/books/missing` و `GET /v2/books/missing` كلاهما:

```json
{ "statusCode": 404, "code": "book_not_found", "message": "Book not found" }
```

سطر الكسر الوحيد في هذه الأمثلة هو شكل `price`.
