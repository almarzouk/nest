# استضافة SPA

## الهدف من هذا الدرس

تقديم بناء واجهة صفحة واحدة من Nest، مع إبقاء مسارات الـ API على رموزها الحقيقية.

## المشكلة التي يحلها

واجهة React أو Vue تطلب `/books/42` في المتصفح حتى يعيد الموجّه على العميل رسم الصفحة. الخادم إن لم يجد متحكمًا بهذا الشكل يُرجع عادة `404`. إن جعلت `ServeStaticModule` يلتقط كل مسار ويرسل `index.html`، اشتغل الموجّه. وإن التقط أيضًا `/api/books/404-no-such-id` أعاد HTML برمز `200`، فانكسر العميل الذي يتوقع JSON ورمز غياب.

## الفكرة ببساطة

`renderPath` الافتراضي بدل يستوعب كل المسارات حتى يصل `index.html` لمسارات الواجهة. قيّد هذا البدل: بادئة للواجهة، أو `exclude` لبادئة الـ API على Express. على Fastify لا يوجد `exclude`. استخدم `renderPath` تعبيرًا منتظمًا. وتحقق بطلب API غائب أن الجسم ليس HTML.

## التعريف التقني

`renderPath` مسار رسم التطبيق الثابت، ويُضم إلى `serveRoot`. الافتراض على Express `{*any}` وعلى Fastify `*`. أي مسار غير مستثنى يمكن أن يُرسل له `index.html` إن كان `index` مفعّلًا و `fallthrough` يسمح بالسقوط إلى ملف الفهرس.

مثال Nest لواجهة مع API:

```ts
ServeStaticModule.forRoot({
  rootPath: join(__dirname, '..', 'client'),
  exclude: ['/api/{*path}'],
  serveStaticOptions: { fallthrough: false },
});
```

`fallthrough: false` في المثال الرسمي يمنع الطبقة من تمرير ما لم تجده. مع استضافة SPA غالبًا تريد العكس لمسارات الواجهة: غياب الملف يعني `index.html` لا `404`. اجمع هذا مع استثناء `/api` حتى غياب JSON يبقى من Nest لا من ملف الفهرس.

`useGlobalPrefix: false` هو الافتراض. إن ناديت `setGlobalPrefix('api')` ثم فعّلت `useGlobalPrefix`، قد تُخدم الواجهة تحت `/api` أيضًا. اتركه `false` عندما تكون البادئة العامة للـ API فقط.

## Mental Model

```text
/api/*     متحكمات، والغياب 404 JSON
/*         أصل إن وُجد، وإلا index.html للواجهة
```

الترتيب قرار إعداد، لا أملًا في أن «المتحكم يفوز دائمًا».

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'node:path';
import { BooksModule } from './books/books.module.js';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(process.cwd(), 'client', 'dist'),
      exclude: ['/api/{*path}'],
    }),
    BooksModule,
  ],
})
export class AppModule {}
```

ومسار الواجهة `GET /books/42` يعيد `index.html`. مسار `GET /api/books/42` يذهب إلى المتحكم لأن الاستثناء يطابق `/api/...`.

## شرح المثال

`client/dist` ناتج بناء الواجهة وفيه `index.html` وملفات مجزّأة بأسماء فيها تجزئة. تلك الملفات أصول حقيقية تُخزَّن مؤقتًا طويلًا. `index.html` وحده يتغير مع كل نشر حتى يشير إلى الأسماء الجديدة، فتخزينه المؤقت قصير. الاستثناء يحمي شجرة `/api` كلها، بما فيها مسار لم تكتبه بعد. من غير الاستثناء، مسار API خاطئ قد يعيد الصفحة.

## ماذا يحدث داخليًا؟

الوسيط يرى الطلب قبل إكمال السلسلة أو ضمنها حسب تركيب Nest. إن طابق `exclude` يتخطى الخدمة الثابتة. وإلا يبحث عن ملف. إن لم يوجد ملف وطبقة SPA معدّة للفهرس، يُرسل `index.html` مع `200`. عميل `fetch` الذي فحص `response.ok` فقط يعتبر الغياب نجاحًا ثم يفشل عند `response.json()` لأن الجسم HTML. اختبار هذا المسار أوضح من قراءة الإعداد مرتين.

على Fastify استبدل الاستثناء بقيد الرسم، لأن الحزمة تصرّح أن `exclude` غير مدعوم هناك:

```ts
renderPath: /^(?!\/api).*/,
```

راجع التعبير مع مساراتك الفعلية. خطأ في النظر الأمامي يعيد فتح الثغرة أو يخفي الواجهة.

## مثال عملي

قائمة تحقق بعد ربط البناء:

| الطلب | الرد المتوقع |
| --- | --- |
| `GET /` | `index.html` |
| `GET /books/42` | `index.html` حتى بلا كتاب 42، لأن الموجّه على العميل |
| `GET /api/books/42` | JSON أو `404` JSON |
| `GET /api/missing` | `404` من Nest، الجسم ليس `<!doctype html>` |
| `GET /assets/index-abc123.js` | الملف المجزّأ |

إن فشل الصف الأخير فـ `rootPath` لا يشير إلى `dist`. إن فشل صف `/api/missing` فالاستثناء لا يطابق صيغة البدل.

## أخطاء شائعة

- `exclude: ['/api*']` على إصدار يستعمل `{*name}`.
- الحكم على الـ API من المتصفح الذي يعرض HTML ولا يظهر رمز `404`.
- تخزين `index.html` لسنة، فيبقى العميل على حزم قديمة.
- تقديم SPA من Nest في الإنتاج وقد وضعت CDN أصلًا. عندها الخادم يكرر الدور ويكشف إعداد البدل.

## متى نستخدمه؟

عندما تنشر الواجهة والـ API من عملية Node واحدة، والموجّه على العميل يحتاج `index.html` لكل مسار شاشة.

## متى لا نستخدمه؟

عندما يخدم Nginx أو CDN الملفات وNest تستقبل `/api` فقط. البدل هناك إعداد الخادم الأمامي. ولا تستخدم البدل إن لم تكن هناك واجهة، لأنه يخفي المسارات الناقصة.

## مقارنة

| الطلب غير الموجود | بلا SPA | مع بدل بلا استثناء | مع استثناء `/api` |
| --- | --- | --- | --- |
| `/books/42` | `404` | `index.html` | `index.html` |
| `/api/nope` | `404` JSON | `index.html` و `200` | `404` JSON |

## تمارين

### تمرين سهل

لماذا تعيد الواجهة `index.html` لمسار `/books/42`؟

<details>
<summary>الحل</summary>

لأن رسم الشاشة يحدث في المتصفح. الخادم يسلّم تطبيق الواجهة لأي مسار شاشة، والبيانات تأتي من `/api`.

</details>

### تمرين متوسط

ما علامة أن استثناء API فشل؟

<details>
<summary>الحل</summary>

`GET /api/missing` يعيد `200` وجسم HTML بدل `404` من Nest.

</details>

### تمرين متقدم

لماذا `exclude` لا يكفي خطة Fastify؟

<details>
<summary>الحل</summary>

الحزمة لا تدعم `exclude` على Fastify. القيد يكون `renderPath` بتعبير منتظم يستثني بادئة الـ API.

</details>

## أسئلة مراجعة

- ما الافتراض الخطير لـ `renderPath`؟
- أين يجلس بناء الواجهة مقابل مجلد الأغلفة؟
- ماذا تفحص غير صفحة البداية؟

## الخلاصة

SPA من Nest تعني `index.html` لمسارات الشاشات، واستثناءً صريحًا لبادئة الـ API حتى يبقى الغياب JSON.

## ماذا بعد؟

GraphQL من الصفر، ثم كيف يولّد Nest المخطط من الأصناف.

## المصادر الرسمية

- `@nestjs/serve-static`: https://github.com/nestjs/serve-static
- مثال الاستثناء: https://github.com/nestjs/nest/blob/master/sample/24-serve-static/src/app.module.ts
