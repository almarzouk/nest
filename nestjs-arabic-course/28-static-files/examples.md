# أمثلة عملية

ثلاثة إعدادات. لا تضعها كلها في التطبيق نفسه. اختر واحدًا حسب ما يجلس على المنفذ.

## مثال 1 — أصول عامة فقط

الـ API على `/` والشعار تحت بادئة. لا بدل يلتقط الموقع.

```ts
import { Module } from '@nestjs/common';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'node:path';
import { BooksModule } from './books/books.module.js';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(process.cwd(), 'public'),
      serveRoot: '/assets',
      serveStaticOptions: {
        index: false,
        fallthrough: false,
        dotfiles: 'ignore',
        maxAge: '1h',
      },
    }),
    BooksModule,
  ],
})
export class AppModule {}
```

`GET /assets/logo.svg` ملف. `GET /assets/../.env` لا يخرج من `public`. `GET /books/1` JSON. مجلد `var/covers` غير مذكور.

## مثال 2 — واجهة و API معًا

```ts
ServeStaticModule.forRoot({
  rootPath: join(process.cwd(), 'client', 'dist'),
  exclude: ['/api/{*path}'],
  serveStaticOptions: {
    maxAge: '1y',
    setHeaders(res, path) {
      if (path.endsWith('index.html')) {
        res.setHeader('Cache-Control', 'no-cache');
      }
    },
  },
});
```

الحزم المجزّأة تُخزَّن طويلًا. `index.html` يُعاد التحقق منه حتى يشير إلى أسماء الملفات الجديدة بعد النشر. `BooksController` يعيش تحت `@Controller('api/books')` ليلتقي الاستثناء.

تحقق:

```bash
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:3000/api/does-not-exist
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:3000/books/42
```

الأول `404` و JSON. الثاني `200` و HTML.

## مثال 3 — Fastify بلا exclude

```ts
ServeStaticModule.forRoot({
  rootPath: join(process.cwd(), 'client', 'dist'),
  renderPath: /^(?!\/api).*/,
});
```

التعبير يرسم الواجهة لكل مسار لا يبدأ بـ `/api`. جرّب `/api` و `/apiv2` حتى لا يبتلع النظر الأمامي بادئة ثانية. إن كانت البادئة `/v1` فعدّل التعبير. لا تنسخ مثال Express وتتوقع الاستثناء نفسه.

## ماذا تتأكد منه

- `var/covers` ليس `rootPath` ولا ابنًا له.
- ملف نقطة داخل `public` لا يُرسل مع `dotfiles: 'ignore'`.
- مسار API الغائب لا يعيد صفحة الواجهة.
