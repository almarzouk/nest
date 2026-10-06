# أمثلة عملية

ثلاثة أسطر من مسار نشر واحد.

## مثال 1 — المسجّل

```ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(AppLogger));
```

سجّل `AppLogger` في الوحدة إن حقنته، أو `new AppLogger()` إن كان بلا اعتماد. `write` يخرج JSON فيه `level` و `time` و `context` و `message` و `requestId`.

## مثال 2 — الطلب

```ts
this.logger.log({
  message: 'request',
  method: 'POST',
  path: '/books/42/publish',
  statusCode: 201,
  durationMs: 36,
  requestId: '3c2f0a2e-7b1d-4e6a-9c11-0a0b0c0d0e0f',
});
```

بلا `body` وبلا استعلام.

## مثال 3 — العامل

```ts
this.logger.error({
  message: 'resize failed',
  bookId: job.data.bookId,
  requestId: job.data.requestId ?? null,
});
```

الحمولة `{ bookId, coverKey, requestId }` من المنتج الذي قرأ `AsyncLocalStorage` داخل `run`.

## ماذا تتأكد منه

- `x-request-id` فيه سطر جديد لا يُقبل.
- `@Cron` يكتب `requestId: null`.
- فشل ISBN يسجّل `isbn` و `code` لا ترويسة المفتاح.
- المعترض لا يبدّل جسم `StreamableFile`.
