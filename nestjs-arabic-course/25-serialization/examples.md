# أمثلة عملية

ثلاثة مخارج لـ API المقالات. الاستيراد ينتهي بـ `.js`.

## مثال 1 — مستخدم عام

`UserResponse` فيه `id` و `email` و `role` ومنشئ ينسخ هذه الثلاثة. `UsersService.findPublic` يستعمل `select` بالأعمدة نفسها. `GET /me` يعيد `new UserResponse(...)` تحت `ClassSerializerInterceptor`. اختبار يرفض وجود `passwordHash` و `$scrypt$`.

## مثال 2 — مقال بلا كيان مؤلف

`PostResponse` فيه `id` و `title` و `published` و `authorId`. القراءة لا تستعمل `include` لصف المستخدم كاملًا. `@ApiOkResponse({ type: PostResponse })` يطابق الجسم. مسودة غير المالك لا تُجلب، لا تُخفى بعد الجلب فقط.

## مثال 3 — خطأ وبريد مكرر

`P2002` يصبح `ConflictException("Email already registered")`. لا `meta` ولا صف. تسجيل الدخول يسجّل `userId` بعد النجاح لا `RegisterDto`. مثال Swagger للبريد `ada@library.test` من غير تجزئة.

## ماذا تتأكد منه

- `GET /me` و `POST /users` يستعملان `UserResponse`.
- كائن `validate` لا يُعاد كما هو إن كان فيه عمود السر.
- `api-json` بلا `passwordHash`.
- ملف تنزيل `StreamableFile` لا يمر على حذف الحقول، لذلك لا تضع سرًا في ملف عام.
