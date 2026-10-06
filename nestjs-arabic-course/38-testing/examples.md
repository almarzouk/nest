# أمثلة عملية

ثلاثة ملفات باختصار. القالب ESM فيستعمل `vi` من `vitest`. على CommonJS استبدل `jest.fn` واترك `TestingModule`.

## مثال 1 — وحدة نشر

```ts
const update = vi.fn(async () => ({ id: '42', status: 'PUBLISHED' }));
const emitAsync = vi.fn(async () => []);
const service = new BooksService({ book: { update } } as never, { emitAsync } as never);
await service.publish('42');
expect(emitAsync).toHaveBeenCalledWith(bookPublished, { bookId: '42' });
```

حالة ثانية: `update` يرمي و `emitAsync` لا يُنادى.

## مثال 2 — رحلة 400

```ts
const app = moduleRef.createNestApplication();
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
await app.init();
await request(app.getHttpServer()).post('/books').send({}).expect(400);
await app.close();
```

الخدمة وهم `create` و `expect(create).not.toHaveBeenCalled()`.

## مثال 3 — قاعدة اختبار

```ts
if (!process.env.DATABASE_URL?.includes('bookstore_test')) {
  throw new Error('refusing to test without bookstore_test');
}
await prisma.book.deleteMany();
```

ثم `publish` الحقيقي وقراءة `PUBLISHED`. الطابور يبقى وهم `add`.

## ماذا تتأكد منه

- لا Jest و Vitest معًا.
- `overrideProvider` قبل `compile`.
- الحمولة إلى `add` فيها `bookId` لا نموذج Prisma.
- لا `listen(3000)` ولا عنوان إنتاج.
