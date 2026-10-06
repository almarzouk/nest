# Hybrid Applications

## الهدف من هذا الدرس

تشغيل HTTP رف مع مستمع رسائل في نفس العملية عبر `connectMicroservice`، وضبط ترتيب الإقلاع ووراثة الإعداد.

## المشكلة التي يحلها

`createMicroservice()` يبني تطبيقًا واحدًا بلا HTTP بجانبه. أو الرسائل تُستهلك قبل `onModuleInit`. أو حارس المنظمة العام على HTTP لا يصل إلى معالج الرسالة فتُقرأ حمولة بلا نفس السياسة، أو يصل فجأة لأن أحدهم افترض الوراثة.

## الفكرة ببساطة

التطبيق الهجين يسمع من أكثر من مصدر: HTTP مع TCP مثلًا، أو أكثر من ناقل. `NestFactory.create` ثم `connectMicroservice` لكل ناقل، ثم إما `listen` أو `init` إن لم يوجد HTTP، و `startAllMicroservices`. الترتيب قرار: بدء الخدمات المصغرة أولًا يستهلك رسائل قبل اكتمال خطافات الحياة. الوراثة `inheritAppConfig` ليست افتراضية. المعالجات تُسجَّل عند `connectMicroservice`، لذا `useGlobal*` قبل الربط إن أردتم الوراثة. الصفحة: https://docs.nestjs.com/faq/hybrid-application

## التعريف التقني

```ts
const app = await NestFactory.create(AppModule);
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.TCP,
});
await app.startAllMicroservices();
await app.listen(3001);
```

`listen` يفتح HTTP. بلا مسارات HTTP استدعوا `app.init()` بدلًا منه. لربط الناقل بعد اكتمال الوحدات:

```ts
await app.listen(3001);
await app.startAllMicroservices();
```

أكثر من مستمع يعني أكثر من `connectMicroservice` (TCP و Redis في مثال الصفحة). `@MessagePattern` يأخذ ناقلًا ثانيًا مثل `Transport.NATS` حين تتعدد النواقل. `inheritAppConfig: true` كوسيط ثانٍ ينسخ الأنابيب والاعتراضات والحراس والمرشحات العامة، سواء سُجّلت بـ `useGlobal*` أو برموز `APP_GUARD` وأخواتها. افتراضيًا الخدمات المتصلة لا ترث ذلك.

## Mental Model

```text
create → connectMicroservice (تسجيل المعالجات)
→ init أو listen
→ startAllMicroservices
الحارس العام لا يُورَّث إلا بعلم صريح
```

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: { port: 3002 },
  });
  await app.listen(3001);
  await app.startAllMicroservices();
}
await bootstrap();
```

`listen` هنا يفتح HTTP ويكمل خطافات الإقلاع قبل الاستهلاك. بلا مسارات HTTP استبدلوه بـ `app.init()` ثم `startAllMicroservices`. بدء الخدمات المصغرة أولًا يوصل الرسائل قبل تلك الخطافات.

## شرح المثال

- منفذ 3001 للمتصفح و 3002 لرسائل TCP. لا تشاركوهما بلا قصد.
- Fastify أو Express يبقى اختيار منصة HTTP كما في بقية رف. Express هو الافتراضي إن لم تقيسوا غيره.
- الرسالة ليست طلبًا فيه ترويسة منظمة.

## ماذا يحدث داخليًا؟

1. `create` يبني تطبيق HTTP ويهيئ الوحدات عند `init` أو `listen`.
2. `connectMicroservice` يربط `INestMicroservice` ويسجّل `@MessagePattern` فورًا.
3. `startAllMicroservices` يبدأ الاستهلاك.
4. إن سبق الاستهلاكُ الخطافات، قد يصل أمر «انشر» قبل جاهزية Prisma.
5. مع `inheritAppConfig` تُنسخ المحسّنات المسجّلة قبل `connect`. نداء `useGlobalGuards` بعده لا يصل إلى معالجات ذلك المستمع.

## مثال عملي

رف تبقي `POST /projects` على HTTP مع حارس المنظمة. عامل داخلي على TCP يستقبل `project.published` الذي وضعه الصندوق. المعالج يحمّل `organizationId` من الحمولة ولا يعتمد على حارس HTTP. إن احتجتم نفس الأنبوب على الرسالة فمرّروا `{ inheritAppConfig: true }` بعد تسجيل الأنابيب العامة. لا تفعّلوا الخطة من رسالة داخلية بلا توقيع المصدر. رسالة التكامل داخل شبكتكم غير حدث Stripe.

## أخطاء شائعة

- `createMicroservice` مع توقع أن `listen` ما زال متاحًا على نفس الكائن.
- افتراض وراثة `TenantGuard` على كل `@MessagePattern`.
- `startAllMicroservices` قبل `init` ومعالج يلمس قاعدة لم تُفتح.
- منفذ TCP على واجهة عامة بلا حاجة.

## متى نستخدمه؟

عملية واحدة تخدم المتصفح وتستهلك طابورًا أو TCPًا قريبًا، مع ترتيب إقلاع واضح.

## متى لا نستخدمه؟

حين يجب أن يتوسع مستهلك الرسائل مستقلًا عن HTTP. افصلوا تطبيق العامل في مساحة العمل. ولا كبديل عن صندوق الصادر: الهجين ينقل الرسالة بعد ما كُتبت.

## مقارنة

| الاستدعاء | الأثر |
| --- | --- |
| `listen` | HTTP |
| `init` | خطافات بلا HTTP |
| `startAllMicroservices` | بدء الاستهلاك |
| `inheritAppConfig: true` | نسخ المحسّنات العامة المسجّلة قبل الربط |

## تمارين

### تمرين سهل

ماذا تستدعون إن لم يوجد HTTP؟

<details>
<summary>الحل</summary>

`app.init()` بدل `listen`. ثم `startAllMicroservices` بالترتيب الذي يناسب جاهزية الوحدات.

</details>

### تمرين متوسط

لماذا الحارس العام غائب عن معالج TCP؟

<details>
<summary>الحل</summary>

الوراثة ليست افتراضية. `inheritAppConfig: true` ينسخها، بشرط تسجيل `useGlobalGuards` قبل `connectMicroservice`.

</details>

### تمرين متقدم

رسالة وصلت قبل اتصال Prisma. أي ترتيب يمنعه؟

<details>
<summary>الحل</summary>

`init` أو `listen` أولًا حتى تكتمل `onModuleInit` و `onApplicationBootstrap`، ثم `startAllMicroservices`. بدء الخدمات المصغرة أولًا يستهلك قبل تلك الخطافات، كما تلاحظ الصفحة.

</details>

## أسئلة مراجعة

- لماذا لا يكفي `createMicroservice`؟
- ما الوسيط الذي يفتح الوراثة؟
- متى تُسجَّل معالجات الرسائل؟
- أي حقل في الحمولة يحدد المنظمة؟

## الخلاصة

التطبيق الهجين في رف هو HTTP زائد `connectMicroservice`. `init` أو `listen` قبل الاستهلاك يحمي الإقلاع. حارس المنظمة لا يعبر إلى الرسالة إلا بـ `inheritAppConfig`، والحمولة ما زالت تحمل معرف المنظمة صراحة.

## ماذا بعد؟

هذا آخر درس في المسار. الميزات هنا اختيارية. العقد والأمان والاعتمادية التي سبقت تبقى أساس النشر.

## المصادر الرسمية

- NestJS — Hybrid application: https://docs.nestjs.com/faq/hybrid-application
- NestJS — Microservices: https://docs.nestjs.com/microservices/basics
