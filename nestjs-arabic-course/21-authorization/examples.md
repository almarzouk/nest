# أمثلة عملية

التفويض على مقالات المكتبة بعد `@nestjs/authentication`. CASL غير مثبّت.

## مثال 1 — سياسة

```ts
@Policy()
export class PostPolicy {
  create(user: PublicUser | null) {
    return !!user?.roles.includes("editor");
  }

  update(user: PublicUser | null, post: { authorId: number }) {
    return !!user && user.id === post.authorId;
  }

  read(user: PublicUser | null, post: { published: boolean; authorId: number }) {
    if (post.published) return true;
    return !!user && user.id === post.authorId;
  }
}
```

`true` فقط عند السماح. المسودة يراها صاحبها. المنشور يراه الجميع بمن فيهم الضيف إن كان المسار اختياري المصادقة.

## مثال 2 — إنشاء على الحارس وتعديل في الخدمة

`POST /posts` عليه `@Can(PostPolicy, "create")`. القارئ يأخذ `403`. `PATCH /posts/:id` يحمّل الصف، `404` إن غاب، ثم `authorize(PostPolicy, "update", user, post)` ثم تحديث العنوان.

## مثال 3 — ترتيب الإقلاع

`AuthenticationModule` ثم `AuthorizationModule.forRoot()` في `AppModule`. `PostPolicy` ضمن `providers` في `PostsModule`.

## ماذا تتأكد منه

- لا `@Can` على `update`.
- الدور لا يأتي من جسم التسجيل ولا من claim قديم.
- اختبار `new PostPolicy().update(...)` بلا قاعدة.
- لا حارس CASL موازٍ.
