# أمثلة عملية

ثلاث زينات على الشباك، كل ملف يصدّر زينة واحدة. الحارس العام يقرأ `@Public` ثم الهوية، وحارس الأدوار يقرأ `@Roles` بعده.

## مثال 1 — المستخدم الحالي

```ts
export const CurrentUser = createParamDecorator(
  (data: keyof DeskUser | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest<{ user?: DeskUser }>();
    return data ? request.user?.[data] : request.user;
  },
);
```

`refund(@CurrentUser('name') name: string)` لا يفتح `@Req()`. الحارس هو من يملأ `user` قبل أن تُبنى الوسائط.

## مثال 2 — أدوار الدالة

```ts
export const Roles = Reflector.createDecorator<string[]>();
```

```ts
@Roles(['refund'])
@Post('refund')
refund(@CurrentUser('name') name: string): { name: string } {
  return { name };
}
```

`RolesGuard` يطابق `user.roles` بعد حارس الهوية. `guest` بلا `refund` يأخذ `403`. مسار بلا `@Roles` يمر من هذا الحارس.

## مثال 3 — صحة معلَّمة

```ts
export const Public = Reflector.createDecorator<boolean>();
```

```ts
@Public(true)
@Get('health')
health(): { ok: true } {
  return { ok: true };
}
```

أول أسطر `IdentifyDeskGuard` حين يكون `APP_GUARD`:

```ts
const isPublic = this.reflector.getAllAndOverride(Public, [
  context.getHandler(),
  context.getClass(),
]);
if (isPublic) {
  return true;
}
```

`GET /health` بلا `x-desk-name` ينجح. `POST /tickets` بلا الترويسة يبقى `401`. `exclude` غير مستخدم.

## ماذا تتأكد منه

- `experimentalDecorators` و `emitDecoratorMetadata` و `import 'reflect-metadata'` كما ولّدها CLI.
- استيراد الزينة قيمة وبلاحقة `.js`.
- `Roles` و `Public` تُقرآن بالمرجع لا بسلسلة منسوخة.
- `@Public(true)` لا يُوضع على متحكم التذاكر.
