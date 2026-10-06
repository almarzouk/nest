# أمثلة عملية

ثلاث نبضات. القص نفسه يبقى في BullMQ.

## مثال 1 — التسجيل

```ts
@Module({
  imports: [ScheduleModule.forRoot(), CoversModule],
  providers: [CleanupService, QueueWatchService],
})
export class AppModule {}
```

`forRoot` مرة واحدة.

## مثال 2 — تقويم ومنطقة

```ts
@Cron('0 0 1 * * *', { timeZone: 'Asia/Amman' })
async queueDailySales() {
  const day = this.clock.dayIn('Asia/Amman');
  await this.reports.add('daily-sales', { day }, { jobId: `sales:${day}` });
}
```

## مثال 3 — تباعد ومرة إقلاع

```ts
@Interval(30_000)
async logWaiting() {
  const counts = await this.covers.getJobCounts('waiting');
  if (counts.waiting > 100) this.logger.warn(`covers waiting ${counts.waiting}`);
}

@Timeout(5_000)
logReady() {
  this.logger.log('scheduler up');
}
```

`logReady` ليس مهلة ISBN. تذكير المسودات يجمع معرفات ويستدعي منتج البريد، ولا يرسل حلقة بريد داخل `@Cron`.

## ماذا تتأكد منه

- الاختبار ينادي الدالة ولا ينتظر الدقيقة 15.
- نسختان لا ترسلان تقريرين بفضل `jobId`.
- لا `@Cron` يقص الأغلفة ملفًا ملفًا.
