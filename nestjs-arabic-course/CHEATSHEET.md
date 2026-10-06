# مرجع سريع

ينمو مع الفصول. هذا الإصدار يغطي المقدمة فقط. التفاصيل والسبب في الدروس، لا في هذا الملف.

## HTTP

| Method | استخدام شائع | تكرار مكافئ |
| --- | --- | --- |
| `GET` | قراءة | لا يغيّر الحالة |
| `POST` | إنشاء أو تشغيل عملية | قد يكرر الأثر |
| `PUT` | استبدال التمثيل كاملًا | يوصل إلى الحالة نفسها |
| `PATCH` | تعديل جزء | يعتمد على شكل التعديل |
| `DELETE` | حذف | الحذف المتكرر يترك المورد غائبًا |

| الرمز | معنى عملي |
| --- | --- |
| `200` | نجاح عام |
| `201` | أُنشئ مورد |
| `204` | نجاح بلا جسم |
| `400` | الطلب غير صالح |
| `401` | فشل Authentication |
| `403` | ممنوع بعد معرفة الهوية |
| `404` | لا مورد بهذا العنوان |
| `409` | يتعارض مع حالة حالية |
| `500` | فشل الخادم |

JSON: مفاتيح بنص مزدوج الاقتباس، `true` و `false` و `null`، ومن غير تعليقات ومن غير فاصلة أخيرة.

## Express

```ts
import express from 'express';

const app = express();
app.use(express.json());
app.get('/books/:id', (req, res) => {
  res.json({ id: req.params.id });
});
app.listen(3000);
```

- `next()` يمرر إلى الطبقة التالية.
- إنهاء `res` من غير `next` يوقف السلسلة.
- نسيان الاثنين يعلّق العميل.
- `req` و `res` هما كائنا Node.

## معاينة Nest

لا تستخدم هذا المرجع بديلًا عن الفصول `02` وما بعدها.

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

المنصة الافتراضية: Express، الحزمة `@nestjs/platform-express`.

Fastify: محوّل `FastifyAdapter` من `@nestjs/platform-fastify`، عندما يصل درس الأداء.

حزم مشروع من غير CLI: `@nestjs/core`، `@nestjs/common`، `rxjs`، `reflect-metadata`.

| Express | Nest |
| --- | --- |
| `app.get('/books', handler)` | `@Controller('books')` و `@Get()` |
| `res.json(value)` داخل المعالج | `return value` في الحالة القياسية |
| `app.use(...)` | Middleware في أول مسار الطلب |
| فحص يدوي للهوية | Guard |
| فحص يدوي للجسم | Pipe |
| `try/catch` مختلف في كل مسار | Exception Filter |

رمز النجاح عند إرجاع قيمة من المتحكم: `200`، و `POST` افتراضيًا `201`. التغيير بـ `@HttpCode` في فصل Controllers.

## Request Lifecycle

```text
Middleware
  -> Guards          (Global ثم Controller ثم Route)
  -> Interceptors    (قبل المتحكم، بالاتجاه نفسه)
  -> Pipes           (Global ثم Controller ثم Route ثم معاملات المسار)
  -> Controller
  -> Service
  -> Interceptors    (بعد المتحكم: Route ثم Controller ثم Global)
  -> Exception Filter عند استثناء غير ممسوك (Route ثم Controller ثم Global)
  -> Response
```

استثناء Middleware: الفلاتر العامة فقط.

## أوامر

```bash
node -v
npm i -g @nestjs/cli
nest new bookstore
npm run start:dev
npm run build
npm run start:prod
npm run lint
npm run format
nest g module books
nest g controller books
nest g service books
```

Node لتشغيل Nest 12: 20.19+ أو 22.12+. مولّد CLI: 22.22.3+ أو 24.15+ أو 26+.

## زينة تتكرر

| الزينة | الدور |
| --- | --- |
| `@Module` | حدود الوحدة |
| `@Controller` | بادئة المسار |
| `@Get` `@Post` `@Put` `@Patch` `@Delete` | الطريقة |
| `@Body` `@Param` `@Query` | استخراج |
| `@Injectable` | مزوّد |
| `@UseGuards` | بوابة |
| `@UseInterceptors` | لف التنفيذ |
| `@UsePipes` | فحص أو تحويل |
| `@Catch` | فلتر استثناء |
| `@HttpCode` | تغيير الرمز الافتراضي |

`POST` الافتراضي `201`، وغيره `200`.

## حزم تُفتح عند فصلها

`@nestjs/config`، `@nestjs/swagger`، `@nestjs/throttler`، `@nestjs/bullmq`، `@nestjs/schedule`، `@nestjs/event-emitter`، `@nestjs/terminus`، `@nestjs/graphql`، `@nestjs/passport` أو `@nestjs/authentication` حسب المسار الذي تختاره، و Prisma للبيانات.
