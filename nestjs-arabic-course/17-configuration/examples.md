# أمثلة عملية

أمثلة فصل الإعداد على مكتبة حي. الأسرار قيم وهمية. لا تنسخ عنوان إنتاج إلى المستودع.

## مثال 1 — ما يُرفع وما يبقى محليًا

`.env.example`:

```text
NODE_ENV=development
PORT=3000
DATABASE_URL=
LIBRARY_NAME=neighborhood-library
```

`.gitignore` يتضمن `.env`. المطوّر ينسخ المثال ويملأ `DATABASE_URL` بعنوان PostgreSQL على جهازه. الخادم يضبط المفاتيح الثلاثة الأولى من لوحة المنصة.

## مثال 2 — مخطط يرفض الإقلاع

```ts
import { z } from "zod";

export const libraryEnv = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.string().min(1),
  LIBRARY_NAME: z.string().min(1).default("neighborhood-library"),
});

export type LibraryEnv = z.infer<typeof libraryEnv>;
```

```ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { libraryEnv } from "./library-env.js";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      cache: true,
      validationSchema: libraryEnv,
    }),
  ],
})
export class AppModule {}
```

بلا `DATABASE_URL` تتوقف `create` وتسمّي الحقل. `PORT=abc` يفشل الإكراه. `LIBRARY_NAME` إن غاب يصبح الاسم الافتراضي.

## مثال 3 — قراءة المنفذ بعد المخطط

```ts
import { NestFactory } from "@nestjs/core";
import { ConfigService } from "@nestjs/config";
import { AppModule } from "./app.module.js";
import type { LibraryEnv } from "./library-env.js";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const config = app.get(ConfigService<LibraryEnv, true>);
  const port = config.get("PORT", { infer: true });
  await app.listen(port);
}

await bootstrap();
```

`infer` يقرأ نوع `PORT` من ناتج Zod. لا تمرّر `Number` مرة ثانية بعد `z.coerce.number()`.

## مثال 4 — نطاق القاعدة ثم مصنع

```ts
import { registerAs } from "@nestjs/config";

export const databaseConfig = registerAs("database", () => ({
  url: process.env.DATABASE_URL ?? "",
}));
```

حمّله في `load` مع `validationSchema` حتى لا يصل `url` فارغًا. معاينة تمرير الاتصال، مع `synchronize: false`، تبقى في [الدرس 8](./08-for-root-async-preview.md). فصل قاعدة البيانات يفتح الاتصال بـ Prisma من `ConfigService.getOrThrow("DATABASE_URL")`.

## ماذا تتأكد منه

- `.env` غير متتبَّع، والمثال بلا أسرار.
- غياب `DATABASE_URL` يفشل قبل `listen`.
- لا توجد قراءة `process.env` داخل `BooksService`.
- سر أو عنوان لا يُطبع في السجل.
