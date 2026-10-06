# أمثلة عملية

قص غلاف بعد إنشاء الكتاب. الحزمة `@nestjs/bullmq` فقط.

## مثال 1 — التسجيل

```ts
BullModule.forRoot({
  connection: { host: 'localhost', port: 6379 },
});

BullModule.registerQueue({ name: coverQueueName });
```

`coverQueueName` هو `'covers'` من `queue-names.ts`.

## مثال 2 — المنتج

```ts
await this.covers.add('resize', { bookId: book.id, coverKey: book.coverKey }, {
  jobId: `resize:${book.id}`,
  attempts: 5,
  backoff: { type: 'exponential', delay: 1000 },
  removeOnFail: false,
});
```

بعد `prisma.book.create` لا قبله. الحمولة بلا نموذج Prisma.

## مثال 3 — المستهلك

```ts
@Processor(coverQueueName)
export class CoversConsumer extends WorkerHost {
  constructor(private readonly covers: CoversService) {
    super();
  }

  async process(job: Job<{ bookId: string; coverKey: string }>) {
    await this.covers.resize(job.data.bookId, job.data.coverKey);
  }
}
```

`resize` تجلب الصف، وتخرج إن اختلف `coverKey`، وتكتب `ready` بعد القص. خطأ القرص يُرمى ليُعاد. الكتاب المحذوف خروج هادئ.

## ماذا تتأكد منه

- لا `@nestjs/bull` ولا `@Process` في الملفات.
- `waitUntilFinished` غير مستخدم في المتحكم.
- بايتات الغلاف ليست في Redis.
- طابور Redis ليس قاعدة مخبأ ISBN إن كنت تمسح المفاتيح في التمارين.
