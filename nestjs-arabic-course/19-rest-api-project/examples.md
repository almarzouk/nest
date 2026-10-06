# أمثلة عملية

مشروع المنشورات على Prisma. المسارات الثلاثة تعمل معًا. لا مصادقة في هذه الأمثلة.

## مثال 1 — مستخدم ثم مقال ثم تعليق

```http
POST /users
{"email":"ada@example.com","name":"Ada","password":"correct-horse"}
```

الاستجابة بلا `passwordHash`:

```json
{"id":1,"email":"ada@example.com","name":"Ada"}
```

```http
POST /posts
{"title":"حدود الوحدة","body":"كل مورد في وحدته","authorId":1}
```

```json
{"id":3,"title":"حدود الوحدة","body":"كل مورد في وحدته","published":false,"authorId":1}
```

```http
POST /posts/3/comments
{"body":"المسودة واضحة"}
```

`authorId` غائب يعيد `404` قبل الإدراج. تعليق على `/posts/999/comments` يعيد `404`.

## مثال 2 — قائمة

```http
GET /posts?published=false&authorId=1&sort=title&order=asc&page=1&pageSize=20
```

`where` يجمع المسودة والمؤلف. الترتيب عنوان ثم `id`. `pageSize=500` يعيد `400`. `"published":"false"` كنص استعلام يجب أن يبقى `false` لا `true`.

## مثال 3 — أخطاء

| الطلب | الرمز |
| --- | --- |
| بريد مكرر | `409` |
| عنوان مقال فارغ | `400` |
| `GET /posts/404040` | `404` |
| `sort=body` | `400` |

انقطاع القاعدة يبقى `500` بلا نص SQL في الجسم.

## ماذا تتأكد منه

- ثلاث وحدات، و `PrismaModule` مشتركة.
- `select` بلا `passwordHash`.
- الأنبوب العام يعمل على الجسم وعلى استعلام القائمة.
- لا حارس هوية بعد، فلا يُنشر المشروع كما هو خارج جهازك.
