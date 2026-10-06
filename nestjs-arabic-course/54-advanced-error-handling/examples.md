# أمثلة عملية

الاستراتيجية على مسار ISBN في رف. الخدمة لا تعرف الوضع، والفلتر لا يعيد البحث في القاعدة.

## المثال 1 — الأصناف

```ts
export class BookNotFound extends Error {
  constructor(readonly bookId: string) {
    super(`missing ${bookId}`);
  }
}

export class IsbnTaken extends Error {
  constructor(readonly isbn: string) {
    super(`taken ${isbn}`);
  }
}
```

## المثال 2 — الخدمة ترمي فقط

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service.js';
import { BookNotFound, IsbnTaken } from './books.errors.js';

@Injectable()
export class IsbnService {
  constructor(private readonly prisma: PrismaService) {}

  async assign(tenantId: string, bookId: string, isbn: string) {
    const book = await this.prisma.book.findFirst({ where: { id: bookId, tenantId } });
    if (!book) throw new BookNotFound(bookId);
    try {
      return await this.prisma.book.update({ where: { id: bookId }, data: { isbn } });
    } catch (error) {
      if (isUniqueViolation(error)) throw new IsbnTaken(isbn);
      throw error;
    }
  }
}

function isUniqueViolation(error: unknown) {
  return typeof error === 'object' && error !== null && 'code' in error && error.code === 'P2002';
}
```

انتهاك التفرد يصبح `IsbnTaken`. أي خطأ Prisma آخر يُعاد رميه ليصل إلى الفلتر الختامي كغير متوقع. لا يوجد `InternalServerErrorException` هنا.

## المثال 3 — الحد

المتحكم:

```ts
@Post(':id/isbn')
assign(@Param('id') id: string, @Body() body: { isbn: string }, @Tenant() tenantId: string) {
  return this.isbn.assign(tenantId, id, body.isbn);
}
```

الفلتر يكتب 404 لـ `BookNotFound` و 409 لـ `IsbnTaken`. اختبار يطلب ISBN مكررًا ويتوقع 409. اختبار ثانٍ يقطع قاعدة البيانات الوهمية بخطأ عام ويتوقع 500 وجسمًا بلا `P1001`. اختبار ثالث يطلب كتابًا بصلاحية حارس ترمي `ForbiddenException` ويتوقع 403 لا 500.
