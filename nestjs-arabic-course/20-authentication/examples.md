# أمثلة عملية

المكتبة على `@nestjs/authentication`. Passport في المثال الأخير مرجع منفصل لا يُستورد معه.

## مثال 1 — تسجيل ودخول بجلسة

`POST /auth/sign-up` و `POST /auth/sign-in` عليهما `@Public()`. كلمة المرور تمر على `PasswordHasher.hash` عند الإنشاء و `verify` عند الدخول. النجاح يستدعي `SignInService.signIn`. الاستجابة لا تحتوي `passwordHash`. فشل الدخول `401` برسالة واحدة.

`AuthenticationModule.forRoot` يضبط `session.absoluteTtl` و `idleTtl`. الإنتاج يربط `SessionStore`، لا ذاكرة العملية.

## مثال 2 — مقال باسم الشخص

`CreatePostDto` فيه `title` و `body` فقط.

```ts
@Post()
create(@CurrentUser() user: PublicUser, @Body() dto: CreatePostDto) {
  return this.posts.create(user.id, dto);
}
```

بلا كعكة: `401` قبل Prisma. مع كعكة المستخدم `4`: الصف يُكتب بـ `authorId` الخاص به ولو أرسل العميل رقمًا زائدًا يرفضه الأنبوب.

## مثال 3 — جوال

`JwtAuth` بترتيب `1` بعد الجلسة. `POST /auth/token` يرجع الوصول والتحديث. `POST /auth/token/refresh` يستهلك التحديث. إعادة الرمز المستهلك تلغي العائلة. `validate` تجلب المستخدم من القاعدة.

## مثال 4 — Passport في ملف لا يُحمَّل

`LocalStrategy` مع `usernameField: "email"` و `JwtAuthGuard` نمط الوصفة إن قرأت مشروعًا قديمًا. `AppModule` للدورة لا يسجّل `APP_GUARD` من `@nestjs/passport` بجانب حارس الحزمة.

## ماذا تتأكد منه

- مسار واحد، حارس واحد.
- السر في الإعداد لا في المستودع.
- `GET /me` بلا تجزئة.
- عمر الحزمة ليس ثواني `@nestjs/jwt` إلا داخل مثال الوصفة.
