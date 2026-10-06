# أمثلة عملية

المكتبة على PostgreSQL و Prisma فقط. TypeORM و Drizzle لا يُسجَّلان في هذا المثال.

## مثال 1 — عنوان من الإعداد

`.env` المحلي فيه `DATABASE_URL` ولا يُرفع. المخطط يرفض الغياب. `PrismaService` تقرأ `getOrThrow` وتبني `PrismaPg`.

```ts
const adapter = new PrismaPg({
  connectionString: config.getOrThrow<string>("DATABASE_URL"),
});
super({ adapter });
```

`onModuleInit` يستدعي `$connect`، و `main.ts` يستدعي `enableShutdownHooks`.

## مثال 2 — مخطط ثم ترحيل

```prisma
model Book {
  id        Int     @id @default(autoincrement())
  title     String
  published Boolean @default(false)
  member    Member? @relation(fields: [memberId], references: [id])
  memberId  Int?
}

model Member {
  id    Int    @id @default(autoincrement())
  email String @unique
  books Book[]
}
```

```bash
npx prisma migrate dev --name init
npx prisma generate
```

الإنتاج لاحقًا: `npx prisma migrate deploy` على الملفات المرفوعة.

## مثال 3 — إعارة في معاملة

```ts
return this.prisma.$transaction(async (tx) => {
  const book = await tx.book.findUnique({ where: { id: bookId } });
  if (!book || book.memberId !== null) {
    throw new Error("Book is not available");
  }
  await tx.loan.create({ data: { bookId, memberId } });
  return tx.book.update({ where: { id: bookId }, data: { memberId } });
});
```

نموذج `Loan` يُضاف بترحيل مستقل قبل هذا الاستدعاء. كل الأثر على `tx`.

## مثال 4 — الوحدة

`PrismaModule` تصدّر `PrismaService`. `BooksModule` و `LoansModule` تستوردانها. المتحكم لا يستدعي `prisma.book` بنفسه.

## ماذا تتأكد منه

- أداة واحدة في `AppModule`.
- `synchronize` غير مستخدم لأن المسار Prisma.
- مجلد `migrations` مرفوع، و `.env` والعميل المولَّد غير مرفوعين.
- اختبار الوحدة يمرّر مزيفًا لـ `PrismaService`.
