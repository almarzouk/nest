# أمثلة عملية

نسختا سعر رف على نفس الخدمة. الفرق في المتحكم والتفعيل فقط.

## المثال 1 — تمثيلان

```ts
export interface ShelfBook {
  id: string;
  title: string;
  priceCents: number;
}

export const toV1 = (book: ShelfBook) => ({
  id: book.id,
  title: book.title,
  price: book.priceCents,
});

export const toV2 = (book: ShelfBook) => ({
  id: book.id,
  title: book.title,
  price: { amount: book.priceCents, currency: 'SAR' },
});
```

## المثال 2 — URI بعد بادئة `api`

```ts
import { VersioningType } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('api');
app.enableVersioning({ type: VersioningType.URI });
await app.listen(process.env.PORT ?? 3000);
```

`GET /api/v1/books/b1` و `GET /api/v2/books/b1`. الصحة على متحكم `VERSION_NEUTRAL` تبقى `GET /api/health` لأن الحياد في URI لا يدرج مقطع النسخة، والبادئة العامة ما زالت تُطبَّق.

## المثال 3 — نفس الكتب بترويسة ثم بـ Accept

لا تفعّل النوعين معًا. اختر واحدًا في `main.ts`.

ترويسة:

```ts
app.enableVersioning({ type: VersioningType.HEADER, header: 'X-Api-Version' });
```

`GET /api/books/b1` مع `X-Api-Version: 2`.

وسائط:

```ts
app.enableVersioning({ type: VersioningType.MEDIA_TYPE, key: 'v=' });
```

`Accept: application/json;v=2` على نفس المسار. اختبار يرسل `v=9` ويتوقع 404. اختبار يرسل النسخة المطابقة ويتوقع `price.amount` في v2 و `price` العدد في v1.
