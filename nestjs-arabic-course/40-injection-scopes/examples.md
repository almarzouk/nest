# أمثلة عملية

## تمرير المستخدم من غير نطاق طلب

```ts
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';

export const CurrentUserId = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest<{ user: { id: string } }>();
    return request.user.id;
  },
);
```

```ts
import { Controller, Get } from '@nestjs/common';
import { BooksService } from './books.service.js';
import { CurrentUserId } from './current-user-id.decorator.js';

@Controller('books')
export class BooksController {
  constructor(private readonly books: BooksService) {}

  @Get()
  list(@CurrentUserId() userId: string) {
    return this.books.list(userId);
  }
}
```

`BooksService` بلا `scope`. Guard سبق وملأ `request.user`. الخدمة Singleton تستقبل `string` وتضيفه إلى `where`.

## عميل قاعدة يبقى DEFAULT

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

تعليم هذا الصنف `REQUEST` يلغي الخطافين ويعيد مجمعًا لكل طلب. الإبقاء على الافتراضي هو المثال الذي يُنسخ.

## مسجّل عابر يعرف أباه

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private readonly parentClass: object) {}

  sayHello(message: string): string {
    const name = this.parentClass?.constructor?.name ?? 'unknown';
    return `${name}: ${message}`;
  }
}
```

`BooksService` و `OrdersService` Singleton تحقنانه فتحصل كل واحدة على نسخة. لا تصعدان إلى نطاق طلب.

## ماذا تتأكد منه

1. طلبان يشاركان مرجع `PrismaService`.
2. `GET /books` لا يعلّم المتحكم `REQUEST` طالما الخدمة لا تحقن مزود طلب.
3. حقن `TenantDb` داخل `BooksService` يجعل المتحكم يُبنى لكل طلب. راقب ذلك قبل الدمج.
4. `HelloService` العابرة تطبع اسم الخدمة التي حقنتها، لا اسمًا واحدًا مشتركًا.
