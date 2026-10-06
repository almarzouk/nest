# أمثلة عملية

ثلاثة تراكيب للوحدات على متجر كتب صغير. الكود الحديث يفترض مشروع ESM كما يولّده NestJS 12، لذلك الاستيراد ينتهي بـ `.js`.

## مثال 1 — ميزة مغلّفة

`BooksModule` تملك الخدمة والمتحكم. جدول الأسعار خاص. الخدمة هي الواجهة الوحيدة.

```ts
@Module({
  controllers: [BooksController],
  providers: [BooksService, PriceTable],
  exports: [BooksService],
})
export class BooksModule {}
```

`PriceTable` تُحقن داخل `BooksService` لأن الاثنين في الوحدة نفسها. وحدة أخرى لا ترى الجدول.

## مثال 2 — نواة تعيد التصدير

التسجيل والتدقيق يُستهلكان معًا.

```ts
@Module({
  imports: [LoggerModule, AuditModule],
  exports: [LoggerModule, AuditModule],
})
export class CoreModule {}
```

`OrdersModule` تستورد `CoreModule` و `BooksModule`. تحقن `BooksService` و `AuditService` إن كان الثاني في `exports` الخاصة بـ `AuditModule`.

## مثال 3 — إعداد مرة واحدة

تحية أو عنوان يمر من الجذر. الميزة لا تعيد `forRoot`.

```ts
@Module({
  imports: [
    GreetingModule.forRoot("أهلًا"),
    CoreModule,
    BooksModule,
    OrdersModule,
  ],
})
export class AppModule {}
```

`NestFactory.create(AppModule)` هو سبب وجود هذه الشجرة: الإطار يبدأ من الجذر ويمشي على `imports`.

## ماذا تتأكد منه

- كل Controller مستخدَم مذكور في `controllers`، لا يكفي وجود الملف.
- كل مزوّد تحقنه وحدة أخرى مذكور في `exports` ومسجّل مرة واحدة.
- `@Global()` غير مستخدم هنا. الاستيراد ظاهر في الجذر والميزات.
