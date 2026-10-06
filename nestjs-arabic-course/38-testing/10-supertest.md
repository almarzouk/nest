# Supertest

## الهدف من هذا الدرس

إرسال طلبات HTTP إلى تطبيق الاختبار عبر `supertest` وقراءة الرمز والجسم، مع `init` و `close`.

## المشكلة التي يحلها

نداء دالة المتحكم لا يمر على المسار ولا على JSON ولا على الأنبوب. متصفح يدوي لا يتكرر في CI. Supertest يستدعي الخادم الذي تبنيه Nest من غير متصفح، وعلى `getHttpServer()` من غير فتح منفذ إن لم تحتجه.

## الفكرة ببساطة

بعد `createNestApplication` و `init`، `request(app.getHttpServer())` يبني الطلب. `expect(201)` يفحص الرمز. `expect({ id: '42' })` يفحص الجسم عندما يكون وهم الخدمة يعيد ذلك. في النهاية `close`.

## التعريف التقني

الحزمة `supertest` وأنواعها للتطوير. صفحة Nest تستعملها للرحلة.

```ts
import request from 'supertest';
import { Test } from '@nestjs/testing';

const books = { create: vi.fn(async () => ({ id: '42', title: 'NestJS' })) };

const moduleRef = await Test.createTestingModule({
  controllers: [BooksController],
  providers: [{ provide: BooksService, useValue: books }],
}).compile();

const app = moduleRef.createNestApplication();
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
await app.init();

await request(app.getHttpServer())
  .post('/books')
  .send({ title: 'NestJS' })
  .expect(201)
  .expect({ id: '42', title: 'NestJS' });

await app.close();
```

`send` يضبط JSON. `post` يطابق `@Post('books')` أو `@Controller('books')` مع `@Post()`. إن كانت البادئة العامة `api` فركّبها في الاختبار كما في `main.ts`: `app.setGlobalPrefix('api')`.

قالب Jest يستعمل `jest.fn` وقد يستورد `supertest` بالأسلوب نفسه. ESM قد يحتاج `import request from 'supertest'` حسب إعداد القالب.

لا تستدعِ `listen(3000)` إلا لعميل خارجي. Supertest لا يحتاجه.

## Mental Model

```text
init
  request(getHttpServer()).post.send.expect
close
```

## أبسط مثال

جسم باطل:

```ts
await request(app.getHttpServer()).post('/books').send({}).expect(400);
expect(books.create).not.toHaveBeenCalled();
```

هذا يثبت الأنبوب على التطبيق الاختباري. إن نُودي `create` فـ `useGlobalPipes` غير مطابق للإنتاج.

## شرح المثال

`expect` الثاني في السلسلة من Supertest يقارن الجسم. `expect` من Vitest ما زال متاحًا للوهم. لا تخلط قراءة الفشل: رمز `400` من Supertest، وعدد النداءات من الوهم. الإثنان معًا يمنعان «400 بعد ما كتبت الصف».

## ماذا يحدث داخليًا؟

`getHttpServer()` يعيد خادم HTTP الخاص بالمحول. Supertest يمرر الطلب في السلسلة: بادئة، أنبوب، حارس، متحكم، وهم. الرد يُسلسَل كأي تطبيق. `close` يوقف الخادم والخطافات. مؤقت `@Interval` إن سُجّل في الوحدة قد يبقي العملية حية حتى `close` أو حتى إلغاء الجدولة. لا تستورد `ScheduleModule` في هذا المصنع.

لملف غلاف، `attach` من Supertest يرسل multipart. أبقِ تلك الرحلة واحدة. بقية حالات الملف وحدة على الأنبوب إن استخرجت التحقق.

## مثال عملي

نشر بلا صلاحية، والحارس حقيقي لا وهم:

```ts
await request(app.getHttpServer()).post('/books/42/publish').expect(403);
```

لا `Authorization`. إن كان الحارس مستبدلًا بـ `overrideGuard` الذي يسمح دائمًا، فالرحلة لا تثبت `403`. اترك الحارس عندما يكون هو السؤال، واستبدل Prisma حتى لا تكتب صفًا.

## أخطاء شائعة

- `listen` على منفذ ثابت.
- نسيان `setGlobalPrefix` المطابق للإنتاج.
- رحلة واحدة تتصل بـ Redis والناشر.
- عدم `close` في `afterAll` عندما يرمي اختبار في الوسط. ضع `close` في `afterAll` لا بعد آخر توقع فقط.

## متى نستخدمه؟

لعقد HTTP: رمز، جسم، ورفض الأنبوب أو الحارس، بعدد صغير من الطلبات.

## متى لا نستخدمه؟

لكل حالة في جدول `mapStatus`. تلك وحدة. Supertest هنا إطار حول التطبيق لا أداة متصفح.

## مقارنة

| النداء | الأنبوب | الرمز |
| --- | --- | --- |
| دالة المتحكم | لا | لا |
| Supertest على `getHttpServer` | نعم إن رُكّب | نعم |
| متصفح | نعم | نعم، وغير متكرر |

## تمارين

### تمرين سهل

أي دالة تعطي Supertest خادم Nest بلا `listen`؟

<details>
<summary>الحل</summary>

`app.getHttpServer()` بعد `init`.

</details>

### تمرين متوسط

لماذا `afterAll` لـ `close` أفضل من سطر في نهاية الاختبار السعيد؟

<details>
<summary>الحل</summary>

لأن الفشل قبل ذلك السطر يترك التطبيق مفتوحًا. `afterAll` يعمل بعد النجاح والفشل.

</details>

### تمرين متقدم

رحلة `403` استبدلت الحارس بآخر يسمح دائمًا. ماذا أثبتت؟

<details>
<summary>الحل</summary>

لا شيء عن المنع. أثبتت المسار مع حارس وهمي. اترك الحارس الحقيقي واستبدل القاعدة.

</details>

## أسئلة مراجعة

- ماذا يضبط `send`؟
- أي إعداد من `main.ts` يجب أن يشاركه المصنع؟
- لماذا لا يُستورد `ScheduleModule` هنا؟

## الخلاصة

Supertest يضرب `getHttpServer()` بعد `init` ويقرأ الرمز والجسم، ثم `close` في `afterAll`. الأنبوب والحارس يُركَّبان كما في الإقلاع، والفروع تبقى في الوحدة.

## ماذا بعد؟

كيف تُفتح قاعدة الاختبار وتُنظَّف من غير لمس الإنتاج.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
- Supertest: https://github.com/ladjs/supertest
