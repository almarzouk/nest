# أمثلة عملية

## forRoot مرة في الجذر

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { DatabaseService } from './database.service.js';

@Module({})
export class DatabaseModule {
  static forRoot(url: string): DynamicModule {
    return {
      module: DatabaseModule,
      global: true,
      providers: [
        { provide: 'DATABASE_URL', useValue: url },
        DatabaseService,
      ],
      exports: [DatabaseService],
    };
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module.js';
import { BooksModule } from './books/books.module.js';

@Module({
  imports: [
    DatabaseModule.forRoot(process.env.DATABASE_URL ?? ''),
    BooksModule,
  ],
})
export class AppModule {}
```

`BooksModule` تحقن `DatabaseService` ولا تستدعي `forRoot`.

## register لسابقتين

```ts
@Module({
  imports: [IdGeneratorModule.register({ prefix: 'usr' })],
  providers: [UsersService],
})
export class UsersModule {}
```

```ts
@Module({
  imports: [IdGeneratorModule.register({ prefix: 'ord' })],
  providers: [OrdersService],
})
export class OrdersModule {}
```

الوحدة غير عامة حتى لا يختلط الرمز بين النداءين.

## إعادة تصدير بلا نداء ثانٍ

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from '../database/database.module.js';

@Module({
  imports: [DatabaseModule.forRoot(process.env.DATABASE_URL ?? '')],
  exports: [DatabaseModule],
})
export class CoreModule {}
```

ميزة لاحقة تستورد `CoreModule` فقط.

## مصنع غير متزامن لقيمة المزود

```ts
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async (config: ConfigService) => ({
    url: config.getOrThrow<string>('DATABASE_URL'),
    ready: true,
  }),
  inject: [ConfigService],
}
```

الخدمة تحقن القيمة بعد انتظار الإطار، وتغلق المورد صنف يملك `onModuleDestroy` إن كان عميلًا حقيقيًا.

## ماذا تتأكد منه

1. بحث في المستودع عن `forRoot(` يظهر مرة للإعداد الواحد.
2. `register` لا يضع `global: true` مع رمز ثابت مشترك.
3. `exports` في النواة هو الصنف لا `DatabaseModule.forRoot(...)`.
4. وحدة التقارير الثقيلة بلا متحكم تُحمَّل بـ `LazyModuleLoader` من عامل، لا من `GET` جديد.
5. `ModuleRef.get` داخل `onModuleInit` لا داخل `constructor`.
