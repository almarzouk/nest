# خريطة الدورة

هذه الخريطة هي ترتيب التعلم. كل سطر فصل كامل داخل مجلده، لا مقالًا يجمع عدة مواضيع كبيرة.

أساس المعلومات: [توثيق NestJS الرسمي](https://docs.nestjs.com/) كما كان منشورًا في 5 أكتوبر 2026، للإصدار الرئيسي **NestJS 12**.

## لماذا هذا الترتيب؟

1. الفصل `00` يبني لغة HTTP و Backend قبل أي Decorator.
2. الفصل `01` يجهّز TypeScript الذي ستقرأه في الملفات المولَّدة، قبل إنشاء المشروع.
3. من `02` إلى `07` نبني الصندوق: تشغيل، Module، Controller، Provider، ثم تفسير Dependency Injection.
4. التوثيق الرسمي يعرض في قسم Overview الترتيب Controllers ثم Providers ثم Modules. نحن نبدأ بـ Modules لأن `NestFactory.create()` يستقبل Root Module، وكل Controller و Provider يُسجَّلان داخله. عند قراءة التوثيق لاحقًا ستجد الموضوعات نفسها بترتيب عرض مختلف.
5. الفصل `08` يرسم Request Lifecycle مرة واحدة، ثم الفصول `09`–`16` تشرح كل محطة.
6. الحد الأدنى من RxJS موضوع داخل بداية فصل Interceptors، لأن `next.handle()` يعيد `Observable`. الفصل `31` يعمّق RxJS قبل `HttpModule`.
7. التحقق يعلّم `ValidationPipe` مع `class-validator`، ثم `StandardSchemaValidationPipe` لأن التوثيق الحالي يوثّق الاثنين معًا.
8. المصادقة تبدأ بمفاهيم الهوية، ثم `@nestjs/authentication` لأنه مسار قسم Security الحالي. وصفة Passport تبقى لأنها ما زالت فصلًا رسميًا في Recipes، وليست مسارًا ملغى.
9. البيانات تشرح الفكرة بلا ORM، ثم PostgreSQL، ثم Prisma كمسار تطبيقي أساسي، مع TypeORM و Drizzle كما يوثّقهما Nest.
10. الطوابير تركز على BullMQ. التوثيق يقول إن Bull في وضع الصيانة بينما BullMQ هو التطوير النشط.
11. الفصول `66` و `67` و `68` إضافات من التوثيق الحالي: الموثوقية، وأدوات CLI، وميزات منصة اختيارية. المسار الأساسي ينتهي وظيفيًا عند المشروع في `65`، وهذه الثلاثة تكمّله.

## المراحل

### 00 — المقدمة: قبل NestJS

- المجلد: [`00-introduction`](./00-introduction/README.md)
- الحالة: مكتمل
- تفهم Backend و HTTP و REST و Node.js و Express، ثم لماذا وُجدت NestJS وكيف تُرتَّب معماريًا.

### 01 — TypeScript الضروري لـ NestJS

- المجلد: [`01-typescript-for-nestjs`](./01-typescript-for-nestjs/README.md)
- الحالة: مكتمل
- كل ما تحتاجه من TypeScript لقراءة كود NestJS وكتابته، بلا دورة لغة كاملة.

### 02 — التثبيت وهيكل المشروع

- المجلد: [`02-project-setup`](./02-project-setup/README.md)
- الحالة: مكتمل
- متطلبات Node.js، و Nest CLI، وإنشاء مشروع، وفهم الملفات التي يولّدها.

### 03 — تشغيل التطبيق

- المجلد: [`03-bootstrap`](./03-bootstrap/README.md)
- الحالة: مكتمل
- ماذا يحدث من NestFactory.create حتى app.listen.

### 04 — Modules

- المجلد: [`04-modules`](./04-modules/README.md)
- الحالة: مكتمل
- كيف تقسّم NestJS التطبيق إلى وحدات ذات حدود واضحة.

### 05 — Controllers

- المجلد: [`05-controllers`](./05-controllers/README.md)
- الحالة: مكتمل
- استقبال HTTP Request وإرجاع Response من غير وضع Business Logic داخل المسار.

### 06 — Providers و Services

- المجلد: [`06-providers`](./06-providers/README.md)
- الحالة: مكتمل
- أين تعيش Business Logic، وكيف تُسجَّل حتى يستطيع Container إنشاءها.

### 07 — Dependency Injection

- المجلد: [`07-dependency-injection`](./07-dependency-injection/README.md)
- الحالة: مكتمل
- لماذا لا ننشئ الاعتماديات بـ new، وكيف يجدها Nest.

### 08 — Request Lifecycle

- المجلد: [`08-request-lifecycle`](./08-request-lifecycle/README.md)
- الحالة: مكتمل
- الترتيب الرسمي لمرور الطلب داخل NestJS.

### 09 — Middleware

- المجلد: [`09-middleware`](./09-middleware/README.md)
- الحالة: مكتمل
- الكود الذي يجري قبل اختيار الـ Route Handler، ومتى يكون هو الأداة المناسبة.

### 10 — Pipes

- المجلد: [`10-pipes`](./10-pipes/README.md)
- الحالة: مكتمل
- تحويل البيانات والتحقق منها قبل أن تصل إلى الـ Handler.

### 11 — DTO

- المجلد: [`11-dto`](./11-dto/README.md)
- الحالة: مكتمل
- شكل البيانات المسموح به على حدود التطبيق.

### 12 — Validation

- المجلد: [`12-validation`](./12-validation/README.md)
- الحالة: مكتمل
- رفض البيانات غير الصالحة قبل Business Logic، بأسلوب Classes وأسلوب Schemas.

### 13 — Guards

- المجلد: [`13-guards`](./13-guards/README.md)
- الحالة: مكتمل
- من يُسمح له بتجاوز هذا المسار: Authentication و Authorization على البوابة.

### 14 — Interceptors

- المجلد: [`14-interceptors`](./14-interceptors/README.md)
- الحالة: مكتمل
- منطق يلتف حول الـ Handler قبل التنفيذ وبعده. يبدأ الفصل بالحد الأدنى من RxJS لأن next.handle يعيد Observable.

### 15 — Exception Filters

- المجلد: [`15-exception-filters`](./15-exception-filters/README.md)
- الحالة: مكتمل
- تحويل الاستثناءات غير الممسوكة إلى Responses مفهومة.

### 16 — Custom Decorators

- المجلد: [`16-custom-decorators`](./16-custom-decorators/README.md)
- الحالة: مكتمل
- بناء @CurrentUser و @Roles و @Public فوق metadata و reflection.

### 17 — Configuration

- المجلد: [`17-configuration`](./17-configuration/README.md)
- الحالة: مكتمل
- إعدادات تختلف بين البيئات، مع التحقق من صحتها.

### 18 — Database

- المجلد: [`18-database`](./18-database/README.md)
- الحالة: مكتمل
- من الاتصال بقاعدة البيانات حتى Prisma كمسار أساسي، مع TypeORM و Drizzle كما يوثّقهما Nest.

### 19 — مشروع REST API

- المجلد: [`19-rest-api-project`](./19-rest-api-project/README.md)
- الحالة: مكتمل
- Users و Posts و Comments تُبنى طبقة بعد طبقة بالمفاهيم السابقة.

### 20 — Authentication

- المجلد: [`20-authentication`](./20-authentication/README.md)
- الحالة: مكتمل
- إثبات هوية المستخدم. المسار الأساسي هو @nestjs/authentication، ثم وصفة Passport الرسمية لأنها ما زالت موثّقة.

### 21 — Authorization

- المجلد: [`21-authorization`](./21-authorization/README.md)
- الحالة: مكتمل
- ماذا يُسمح للمستخدم أن يفعل بعد أن نعرف من هو.

### 22 — Cookies و Sessions

- المجلد: [`22-cookies-and-sessions`](./22-cookies-and-sessions/README.md)
- الحالة: مكتمل
- حالة عبر طلبات HTTP، والفرق العملي بين Cookie Session و JWT.

### 23 — CORS و Security

- المجلد: [`23-cors-and-security`](./23-cors-and-security/README.md)
- الحالة: مكتمل
- حدود المتصفح، والهجمات الشائعة، وما يوفّره Nest جاهزًا.

### 24 — توثيق الـ API

- المجلد: [`24-api-documentation`](./24-api-documentation/README.md)
- الحالة: مكتمل
- OpenAPI عبر @nestjs/swagger حتى يصبح العقد قابلًا للقراءة والتجربة.

### 25 — Serialization

- المجلد: [`25-serialization`](./25-serialization/README.md)
- الحالة: مكتمل
- ما الذي يخرج في الاستجابة، وما الذي يبقى داخل الخادم.

### 26 — Pagination و Filtering و Sorting

- المجلد: [`26-pagination-filtering-sorting`](./26-pagination-filtering-sorting/README.md)
- الحالة: مكتمل
- نمط قابل لإعادة الاستخدام لقوائم REST.

### 27 — File Upload

- المجلد: [`27-file-upload`](./27-file-upload/README.md)
- الحالة: مكتمل
- استقبال الملفات والتحقق منها قبل تخزينها.

### 28 — Static Files

- المجلد: [`28-static-files`](./28-static-files/README.md)
- الحالة: مكتمل
- تقديم ملفات جاهزة عبر ServeStaticModule، ومتى لا يكون هذا عمل الـ API.

### 29 — GraphQL

- المجلد: [`29-graphql`](./29-graphql/README.md)
- الحالة: مكتمل
- GraphQL من الصفر، ثم Code First داخل NestJS.

### 30 — WebSockets

- المجلد: [`30-websockets`](./30-websockets/README.md)
- الحالة: مكتمل
- اتصال يبقى مفتوحًا، و Gateway بدل Controller.

### 31 — RxJS في NestJS

- المجلد: [`31-rxjs`](./31-rxjs/README.md)
- الحالة: مكتمل
- القدر الذي تحتاجه Interceptors و HttpModule: Observable و pipe والمعاملات الأساسية.

### 32 — HTTP Client

- المجلد: [`32-http-client`](./32-http-client/README.md)
- الحالة: مكتمل
- استدعاء API خارجية من داخل Nest.

### 33 — Caching

- المجلد: [`33-caching`](./33-caching/README.md)
- الحالة: مكتمل
- متى نحتفظ بنتيجة، ومتى تصبح النسخة المحفوظة خطرًا.

### 34 — Queues

- المجلد: [`34-queues`](./34-queues/README.md)
- الحالة: مكتمل
- عمل خلفي عبر BullMQ. Bull موثّق أيضًا لكنه في وضع الصيانة.

### 35 — Scheduling

- المجلد: [`35-scheduling`](./35-scheduling/README.md)
- الحالة: مكتمل
- عمل يشتغل بالوقت: Cron و Interval و Timeout.

### 36 — Events

- المجلد: [`36-events`](./36-events/README.md)
- الحالة: مكتمل
- إعلان أن شيئًا حدث، من غير أن يعرف المُعلن كل من سيتصرف.

### 37 — Logging

- المجلد: [`37-logging`](./37-logging/README.md)
- الحالة: مكتمل
- سجل تستطيع قراءته أثناء التطوير وبعد الإطلاق.

### 38 — Testing

- المجلد: [`38-testing`](./38-testing/README.md)
- الحالة: مكتمل
- Unit و Integration و E2E، مع TestingModule و Supertest.

### 39 — Application Lifecycle

- المجلد: [`39-application-lifecycle`](./39-application-lifecycle/README.md)
- الحالة: مكتمل
- لحظات الإقلاع والإيقاف التي يعطيك Nest خطافات لها.

### 40 — Injection Scopes

- المجلد: [`40-injection-scopes`](./40-injection-scopes/README.md)
- الحالة: مكتمل
- DEFAULT و REQUEST و TRANSIENT، وكلفة كل واحد.

### 41 — Circular Dependencies

- المجلد: [`41-circular-dependencies`](./41-circular-dependencies/README.md)
- الحالة: مكتمل
- دورة اعتماد، ومتى تكون عيب تصميم، وماذا يفعل forwardRef.

### 42 — Dynamic Modules

- المجلد: [`42-dynamic-modules`](./42-dynamic-modules/README.md)
- الحالة: مكتمل
- Module يُكوَّن عند الاستيراد: forRoot و register وأشكالها غير المتزامنة.

### 43 — Discovery و Metadata

- المجلد: [`43-discovery-and-metadata`](./43-discovery-and-metadata/README.md)
- الحالة: مكتمل
- قراءة ما كتبه الـ Decorators أثناء التشغيل.

### 44 — Custom Providers المتقدمة

- المجلد: [`44-advanced-custom-providers`](./44-advanced-custom-providers/README.md)
- الحالة: مكتمل
- useClass و useValue و useFactory و useExisting و Symbols.

### 45 — Architecture

- المجلد: [`45-architecture`](./45-architecture/README.md)
- الحالة: مكتمل
- طبقات وحدود، بلا تعقيد زائد عن حاجة NestJS.

### 46 — SOLID داخل NestJS

- المجلد: [`46-solid`](./46-solid/README.md)
- الحالة: مكتمل
- المبادئ الخمسة على Providers و Modules حقيقية.

### 47 — Repository Pattern

- المجلد: [`47-repository-pattern`](./47-repository-pattern/README.md)
- الحالة: مكتمل
- الفرق بين Repository الخاص بـ ORM وحدود الـ Domain.

### 48 — CQRS

- المجلد: [`48-cqrs`](./48-cqrs/README.md)
- الحالة: مكتمل
- @nestjs/cqrs: Command و Query و Event، ومتى يصبح النمط أثقل من المشكلة.

### 49 — Microservices

- المجلد: [`49-microservices`](./49-microservices/README.md)
- الحالة: مكتمل
- من Monolith إلى رسائل بين عمليات، بالنواقل التي يوثّقها Nest.

### 50 — Docker

- المجلد: [`50-docker`](./50-docker/README.md)
- الحالة: مكتمل
- تشغيل NestJS مع PostgreSQL و Redis في بيئة قابلة للتكرار.

### 51 — Production

- المجلد: [`51-production`](./51-production/README.md)
- الحالة: مكتمل
- بناء التطبيق وإيقافه بأمان ووضعه خلف عكس الوكيل.

### 52 — Health Checks

- المجلد: [`52-health-checks`](./52-health-checks/README.md)
- الحالة: مكتمل
- هل العملية حية، وهل هي جاهزة لاستقبال العمل.

### 53 — Performance

- المجلد: [`53-performance`](./53-performance/README.md)
- الحالة: مكتمل
- ما الذي يبطئ Node process، وما الذي يستحق القياس فعلًا.

### 54 — معالجة أخطاء متقدمة

- المجلد: [`54-advanced-error-handling`](./54-advanced-error-handling/README.md)
- الحالة: مكتمل
- استراتيجية واحدة للأخطاء من الـ Domain حتى الـ HTTP Response.

### 55 — API Versioning

- المجلد: [`55-api-versioning`](./55-api-versioning/README.md)
- الحالة: مكتمل
- تشغيل أكثر من نسخة للمسارات داخل التطبيق نفسه.

### 56 — تصميم REST API

- المجلد: [`56-rest-api-design`](./56-rest-api-design/README.md)
- الحالة: مكتمل
- أسماء الموارد، والأكواد، و Idempotency، وعقد الأخطاء.

### 57 — GraphQL المتقدم

- المجلد: [`57-graphql-advanced`](./57-graphql-advanced/README.md)
- الحالة: مكتمل
- حماية المخطط من الاستعلامات المكلفة، وأساسيات Federation.

### 58 — Authentication المتقدم

- المجلد: [`58-advanced-authentication`](./58-advanced-authentication/README.md)
- الحالة: مكتمل
- OAuth 2.0 و OpenID Connect وتدوير التوكن وإبطاله.

### 59 — Multi-tenancy

- المجلد: [`59-multi-tenancy`](./59-multi-tenancy/README.md)
- الحالة: مكتمل
- عزل بيانات العملاء الثلاثة الشائعة، وكيف يُحدَّد المستأجر في الطلب.

### 60 — معمارية Backend لـ SaaS

- المجلد: [`60-saas-backend-architecture`](./60-saas-backend-architecture/README.md)
- الحالة: مكتمل
- خريطة منظمة وعضوية وأدوار واشتراك وفوترة قبل المشروع النهائي.

### 61 — Webhooks

- المجلد: [`61-webhooks`](./61-webhooks/README.md)
- الحالة: مكتمل
- إشعار داخل إلى نظامك، وإشعار خارج منه، مع التوقيع والتكرار.

### 62 — معمارية تكامل Stripe

- المجلد: [`62-stripe-architecture`](./62-stripe-architecture/README.md)
- الحالة: مكتمل
- حدود الخدمة حول Checkout والاشتراك و Webhook، لا دورة Stripe كاملة.

### 63 — Event-Driven Architecture

- المجلد: [`63-event-driven-architecture`](./63-event-driven-architecture/README.md)
- الحالة: مكتمل
- أحداث النطاق وأحداث التكامل والاتساق المؤجل.

### 64 — Observability

- المجلد: [`64-observability`](./64-observability/README.md)
- الحالة: مكتمل
- Logs و Metrics و Traces، ثم ما يوفّره NestJS Observe.

### 65 — المشروع الإنتاجي النهائي

- المجلد: [`65-final-production-project`](./65-final-production-project/README.md)
- الحالة: مكتمل
- SaaS API يُبنى على مراحل ويجمع REST و GraphQL و Prisma و Redis و الاختبارات.

### 66 — Reliability

- المجلد: [`66-reliability`](./66-reliability/README.md)
- الحالة: مكتمل
- فصل أُضيف لأن التوثيق الحالي يفرد resiliency و idempotency و outbox و workflows و locks.

### 67 — CLI و Devtools و Standalone

- المجلد: [`67-cli-devtools-standalone`](./67-cli-devtools-standalone/README.md)
- الحالة: مكتمل
- أدوات العمل اليومي بعد أن تصبح المعمارية مفهومة.

### 68 — ميزات منصة اختيارية

- المجلد: [`68-optional-platform-features`](./68-optional-platform-features/README.md)
- الحالة: مكتمل
- فصول رسمية مفيدة ليست شرطًا لمسار التعلم الأساسي: i18n و Mail و File Storage و SSE و Serverless.

## ماذا تقرأ من التوثيق وأنت تتقدم؟

كل درس مكتوب يختم بروابط الصفحات التي اعتُمد عليها. ابدأ دائمًا من [docs.nestjs.com](https://docs.nestjs.com/). فهرس الفصول بصيغة مناسبة للقراءة الآلية موجود عند [docs.nestjs.com/llms.txt](https://docs.nestjs.com/llms.txt).

## خارج المسار الإلزامي

MongoDB و Sequelize و MikroORM موثّقة رسميًا. لن نمر عليها درسًا درسًا في المسار الأساسي حتى لا تتفرع قبل إتقان Prisma. تُذكر في فصل قاعدة البيانات كبدائل موجودة.
