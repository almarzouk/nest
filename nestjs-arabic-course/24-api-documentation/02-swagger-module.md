# SwaggerModule

## الهدف من هذا الدرس

توليد وثيقة المكتبة وخدمة Swagger UI من تطبيق Nest عبر `SwaggerModule` و `DocumentBuilder`.

## المشكلة التي يحلها

كتابة `openapi.yaml` يدويًا تتأخر عن المتحكمات. فريق يفتح `/api` فيجد صفحة فارغة لأن `setup` لم يُستدعَ، أو لأن Fastify بلا `@fastify/static`. أو تُبنى الوثيقة قبل `enableVersioning` فتُفقد البادئة من المسارات.

## الفكرة ببساطة

`DocumentBuilder` يملأ عنوان الوثيقة ووصفها وإصدارها. `SwaggerModule.createDocument` يقرأ التطبيق ويبني الكائن. غلّف الاستدعاء في دالة حتى يُبنى المستند عند أول طلب. `SwaggerModule.setup` يقدّم الواجهة على مسار مثل `api`.

## التعريف التقني

```bash
npm install --save @nestjs/swagger
```

على Fastify ثبّت `@fastify/static` أيضًا، كما تحذّر صفحة المقدمة.

```ts
import { NestFactory } from "@nestjs/core";
import { DocumentBuilder, SwaggerModule } from "@nestjs/swagger";
import { AppModule } from "./app.module.js";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const config = new DocumentBuilder()
    .setTitle("Library posts")
    .setDescription("Users, posts, and comments")
    .setVersion("1.0")
    .build();
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup("api", app, documentFactory);
  await app.listen(3000);
}
await bootstrap();
```

المصنع يؤخر التوليد حتى أول فتح، فيختصر بدء التشغيل. الصفحة تقول إن الاستدعاء المبكر لـ `createDocument` يحتاج أن يسبق `enableVersioning` حتى تظهر بادئة الإصدار. المصنع لا يتأثر لأن البناء عند الطلب.

`GET /api` الواجهة. `GET /api-json` الوثيقة. غيّر مسار JSON بـ `jsonDocumentUrl` في الوسيط الرابع لـ `setup`.

خيارات الوثيقة الثالثة تشمل `include` و `operationIdFactory` و `ignoreGlobalPrefix` و `deepScanRoutes`. مصنع يعيد `methodKey` فقط يجعل `operationId` مثل `create` بدل `PostsController_create`.

`ui: false` يوقف الصفحة ويبقي التعريف. `raw: ['json']` يحد الصيغ. تعطيل الواجهة لا يعطل JSON، والعكس موثّق في الصفحة.

`useSecurityHeaders` الافتراضي يعمل مع Swagger UI حسب تلميح المقدمة. تخصيص CSP أو Helmet المباشر على Fastify قد يحجب السكربت. الصفحة تعرض توجيهات لذلك الظرف، منها مصادر `validator.swagger.io` للصور.

لا تضع سر جلسة داخل `setDescription`.

## Mental Model

```text
create(AppModule)
  DocumentBuilder عنوان
  documentFactory عند أول طلب
  setup('api') صفحة + JSON
listen
```

## أبسط مثال

```ts
const documentFactory = () =>
  SwaggerModule.createDocument(app, config, {
    operationIdFactory: (_controllerKey, methodKey) => methodKey,
  });
```

## شرح المثال

عملاء التوليد يحصلون على `findAll` بدل اسم طويل. الأسماء تتصادم إن تكررت الطريقة في متحكمين. للمكتبة اجعل المصنع يضم المورد: `posts_findAll`. الثبات أهم من القصر. بعد تغيير المصنع أعد توليد العميل.

`setTitle` يظهر أعلى الصفحة. اجعله اسم المنتج لا اسم المجلد.

## ماذا يحدث داخليًا؟

`createDocument` يمشي المسارات المسجّلة في التطبيق، يقرأ معاملات `@Body` و `@Query` و `@Param`، ويبني `paths`. من غير `@ApiProperty` أو إضافة CLI تبقى خصائص الصنف غائبة عن النموذج رغم وجودها في TypeScript. ذلك درس DTO. الواجهة تطلب JSON ثم ترسل التجربة إلى نفس الخادم.

`patchDocumentOnRequest` في خيارات `setup` يعدّل الكائن قبل الإرسال. مناسب لإخفاء مسار داخلي عن وثيقة عامة. لا تخفِ المسار من الوثيقة وتتركه بلا حارس.

## مثال عملي

شغّل التطبيق وافتح `http://localhost:3000/api`. سترى وسومًا من أسماء المتحكمات لأن `autoTagControllers` افتراضيًا `true`. `PostsController` يصبح وسم `Posts`. عطّله فقط إن كنت تضع `@ApiTags` بنفسك في كل متحكم.

احفظ `api-json` في أداة العميل وقارن `POST /posts` بـ `CreatePostDto`. إن كان النموذج بلا خصائص، الدرس الرابع أو إضافة CLI يملأها. على Fastify، غياب `@fastify/static` يمنع الأصول لا توليد JSON بالضرورة. افحص الرسالة عند فتح `/api`.

## أخطاء شائعة

- نسيان `@fastify/static` على Fastify.
- بناء الوثيقة مرة عند الإقلاع ثم تفعيل الإصدارات بعده.
- `operationId` مكرر بين `UsersController.create` و `PostsController.create` بعد مصنع يبقي اسم الطريقة فقط.
- CSP مخصصة تحجب واجهة Swagger ثم إطفاء السياسة كلها.

## متى نستخدمه؟

في التطوير وعلى بيئة يستطيع الفريق تجربة API فيها. في الإنتاج يمكن إبقاء JSON لشريك مع `ui` حسب سياسة النشر.

## متى لا نستخدمه؟

كبوابة مصادقة. معرفة شكل الجسم ليست سرًا بديلًا عن الجلسة. ولا تستضف الواجهة على الإنترنت بلا قرار، إن كانت التجارب تضرب قاعدة الإنتاج.

## مقارنة

| مصنع مؤجّل | `createDocument` فورًا |
| --- | --- |
| يُبنى عند أول طلب | يُبنى عند الإقلاع |
| يرى الإصدار إن فُعّل قبل الطلب | يحتاج الترتيب قبل الاستدعاء |
| أخف عند البدء | جاهز قبل أي طلب |

## تمارين

### تمرين سهل

أي مسار يفتح الواجهة بعد `setup('api', ...)`؟

<details>
<summary>الحل</summary>

`/api`، والوثيقة على `/api-json` ما لم تغيّر `jsonDocumentUrl`.

</details>

### تمرين متوسط

لماذا يُغلَّف `createDocument` في دالة؟

<details>
<summary>الحل</summary>

لتأخير البناء حتى الطلب، ولتجنّب فقدان بادئة الإصدار التي تحدث مع الاستدعاء المبكر قبل `enableVersioning`.

</details>

### تمرين متقدم

تريد `operationId` فريدًا لكل مورد من غير الاسم الافتراضي الطويل. ماذا يعيد المصنع؟

<details>
<summary>الحل</summary>

سلسلة تضم المتحكم والطريقة بطريقة ثابتة، مثل `posts_create`، لا `methodKey` وحده إن تكررت الأسماء.

</details>

## أسئلة مراجعة

- ماذا تثبّت زيادة على Fastify؟
- أين يُضبط عنوان الوثيقة؟
- هل الواجهة تستبدل `ValidationPipe`؟

## الخلاصة

`DocumentBuilder` ثم مصنع `createDocument` ثم `setup` ينشرون العقد على `/api`. الخصائص داخل DTO تحتاج مزخرفًا أو إضافة، وذلك ما يلي.

## ماذا بعد؟

[Decorators الخاصة بالعمليات](./03-operation-decorators.md) الوسوم والاستجابات.

## المصادر الرسمية

- NestJS — OpenAPI: https://docs.nestjs.com/openapi/introduction
