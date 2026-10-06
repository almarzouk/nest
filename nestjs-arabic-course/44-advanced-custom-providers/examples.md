# أمثلة عملية

## تبديل التنفيذ بالرمز نفسه

```ts
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};
```

المتحكم يحقن `ConfigService` في البيئتين.

## مزيف اختبار

```ts
await Test.createTestingModule({
  controllers: [BooksController],
  providers: [
    {
      provide: BooksService,
      useValue: { findAll: () => [{ id: 'b1', title: 'مقدمة' }] },
    },
  ],
}).compile();
```

لا قاعدة. الدالة المطلوبة فقط مكتوبة على المزيف.

## مصنع واتصال

```ts
{
  provide: 'CONNECTION',
  useFactory: (options: OptionsProvider) => new DatabaseConnection(options.get()),
  inject: [OptionsProvider],
}
```

## لقب للنسخة نفسها

```ts
{ provide: 'LOGGER', useExisting: LoggerService }
```

مع `providers: [LoggerService, alias]`. في Singleton المرجع واحد.

## رمز Symbol لعقد

```ts
export const BOOKS = Symbol('BOOKS');
```

```ts
{ provide: BOOKS, useClass: PrismaBooksAdapter }
```

```ts
constructor(@Inject(BOOKS) private readonly books: BooksPort) {}
```

الواجهة `BooksPort` للنوع. الاختبار يغطي `BOOKS` بـ `useValue`.

## ماذا تتأكد منه

1. التصدير يذكر الرمز الذي يُحقن خارجًا، بما فيه `Symbol` والسلسلة.
2. `useClass` المكرر ليس لقبًا. اللقب `useExisting`.
3. مصنع `async` يُغطى في الاختبار حتى لا يفتح شبكة.
4. لا `provide` على واجهة ممحية.
