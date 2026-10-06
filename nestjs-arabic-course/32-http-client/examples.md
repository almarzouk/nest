# أمثلة عملية

عميل ISBN واحد خلف `IsbnService`. Axios عبر `@nestjs/axios`. المجال يرى وعدًا.

## مثال 1 — الوحدة

```ts
HttpModule.registerAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    baseURL: config.getOrThrow<string>('ISBN_BASE_URL'),
    timeout: Number(config.get<string>('ISBN_TIMEOUT_MS') ?? 800),
    maxRedirects: 2,
    headers: {
      accept: 'application/json',
      authorization: `Bearer ${config.getOrThrow<string>('ISBN_TOKEN')}`,
    },
  }),
});
```

## مثال 2 — الترجمة

```ts
async lookup(isbn: string): Promise<{ title: string } | null> {
  try {
    const response = await firstValueFrom(
      this.http.get<{ title?: string }>(`/isbns/${encodeURIComponent(isbn)}`),
    );
    if (typeof response.data?.title !== 'string') {
      throw new BadGatewayException('isbn payload missing title');
    }
    return { title: response.data.title };
  } catch (error) {
    if (error instanceof BadGatewayException) throw error;
    const axiosError = error as AxiosError;
    if (axiosError.code === 'ECONNABORTED' || !axiosError.response) {
      throw new GatewayTimeoutException('isbn timed out');
    }
    if (axiosError.response.status === 404) return null;
    throw new BadGatewayException('isbn upstream failed');
  }
}
```

## مثال 3 — الاستعمال الاختياري

```ts
let title = input.title;
if (!title && input.isbn) {
  title = (await this.isbn.lookup(input.isbn))?.title;
}
```

إن رمت `lookup` بمهلة، الإنشاء يتوقف. من يريد الإكمال عند العطل يلتقط `GatewayTimeoutException` حول هذا السطر فقط، ويبقي `null` لـ `404`.

## ماذا تتأكد منه

- لا `axios.get` في المتحكم.
- لا طباعة لترويسة `authorization`.
- `404` الخارجي لا يصبح `NotFoundException` لصفك.
- اختبار الوحدة يستبدل `IsbnService` ولا يتصل بالناشر.
