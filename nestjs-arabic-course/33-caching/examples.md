# أمثلة عملية

عنوان ISBN في Redis مشترك. صلاحية المسودة تبقى استعلامًا.

## مثال 1 — التسجيل

```ts
CacheModule.registerAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    stores: [createKeyv(config.getOrThrow<string>('REDIS_URL'))],
  }),
});
```

في اختبار الوحدة لا تضع `createKeyv`. حقن `{ get: async () => undefined, set: async () => undefined, del: async () => undefined }`.

## مثال 2 — lookup

```ts
async lookup(isbn: string): Promise<{ title: string } | null> {
  const key = `isbn:${isbn}`;
  const cached = await this.cache.get<{ title: string } | null>(key);
  if (cached !== undefined) return cached;
  const fresh = await this.fetchFromPublisher(isbn);
  const ttl = fresh === null ? 60_000 : 10 * 60 * 1000;
  await this.cache.set(key, fresh, ttl);
  return fresh;
}
```

`fetchFromPublisher` ترمي عند المهلة، فلن يُستدعى `set`.

## مثال 3 — ما لا يُكتب

```ts
async canReadDraft(bookId: string, userId: string) {
  return this.membership.has(bookId, userId);
}
```

بلا `cache.set`. مفتاح `draft:${bookId}` مرفوض في المراجعة.

## ماذا تتأكد منه

- `ttl` بالميلي ثانية.
- `null` المخزّن لا يُعاد جلبه في كل ضغطة.
- فشل Redis حول `get` فقط لا يبتلع `GatewayTimeoutException`.
- الاستيراد من `@nestjs/cache-manager`.
