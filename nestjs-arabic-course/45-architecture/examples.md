# أمثلة عملية

## طبقات داخل وحدة

`BooksController` ينادي `BooksService.create(dto)` والخدمة تنادي `prisma.book.create`. المتحكم بلا عميل. مهمة طابور لاحقة تنادي الخدمة نفسها.

## حد تصدير

```ts
@Module({
  controllers: [BooksController],
  providers: [BooksService],
  exports: [BooksService],
})
export class BooksModule {}
```

حذف `exports` يكسر حقن الطلبات. هذا حد رخيص يبقى في المشروع الصغير.

## منفذ عندما يظهر محول ثانٍ

`CreateBook` يحقن `BOOKS` من نوع `BooksPort`. `PrismaBooksAdapter` في `useClass`. اختبار `new CreateBook(fake)` بلا HTTP. لا تضع هذا القالب على `GET /health`.

## سياق لغة

`Money` بلا معرف يرفض السالب. `Order.addLine` بوابة التجميع. الكتالوج لا يكتب سطور الطلب. العبور `bookId`.

## ماذا تتأكد منه

1. لا `prisma` في الملفات `*.controller.ts`.
2. لا `@Global()` على وحدة ميزة.
3. منفذ جديد مربوط بمحول ثانٍ أو باختبار سياسة، لا بكل مسار.
4. كلمتان بمعنى مختلف تعيشان في وحدتين.
