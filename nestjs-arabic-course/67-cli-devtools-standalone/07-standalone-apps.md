# Standalone Apps

## الهدف من هذا الدرس

تشغيل حاوية رف بلا مستمع شبكة عبر `NestFactory.createApplicationContext`، لمهمة تصدير أو سكربت ينتهي ثم يغلق.

## المشكلة التي يحلها

تصدير الفواتير يفتح `listen(3000)` بلا داعٍ فيحجز منفذًا. أو السكربت يستدعي الخدمة بـ `new` فيتجاوز الوحدات. أو العملية تبقى معلقة بعد انتهاء العمل لأن `close()` لم يُستدعَ.

## الفكرة ببساطة

التطبيق المستقل غلاف حول حاوية الحقن. `createApplicationContext(AppModule)` يبني المزودات بلا HTTP. `app.get(InvoiceExportJob)` يجلب النسخة. `app.select(BillingModule).get(..., { strict: true })` يحصر البحث في وحدة. في النهاية `await app.close()` يطلق خطافات الإيقاف ويخرج. الحراس والاعتراضات والأنابيب لا تلتف حول استدعائكم المباشر.

## التعريف التقني

من صفحة Standalone applications:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  await app.close();
}
await bootstrap();
```

`get` يجد متحكمًا أو مزودًا من الوحدات المسجّلة. بلا `strict` الجذر هو المحدد. `select` يمشي في الشجرة. للوحدة الديناميكية مرّروا نفس كائن `register` الذي استوردته `AppModule`، لا استدعاء `register` جديدًا. الصفحة تذكر المهام المجدولة و CLI فوق هذا الغلاف. `nest-commander` في الدرس التالي يبني أوامر فوق الفكرة نفسها.

## Mental Model

```text
createApplicationContext → مزودات بلا منفذ
get(المهمة) → وسيط المنظمة صريح
close → خطافات الإيقاف وخروج العملية
```

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';
import { InvoiceExportJob } from './billing/invoice-export.job.js';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  const job = app.get(InvoiceExportJob);
  await job.exportDay();
  await app.close();
}
await bootstrap();
```

## شرح المثال

- لا `listen`. لا مسار نجاح في المتصفح.
- `exportDay` يمر على المنظمات من القاعدة أو من وسيط السكربت. لا يقرأ ترويسة غير موجودة.
- `close` يغلق اتصال Prisma إن كانت الوحدة تربط الإغلاق بخطاف الإيقاف.

## ماذا يحدث داخليًا؟

1. المصنع يهيئ الوحدات ويستدعي `onModuleInit` كما في تطبيق HTTP.
2. لا محول Express ولا Fastify فلا وسيط منصة ولا أنابيب مسار.
3. `get` يبحث عن الرمز. نسخة ثانية من وحدة ديناميكية لا تطابق الكائن المسجّل فيفشل `select`.
4. استدعاء تابع على الكائن لا يمر بحارس `TenantGuard` حتى لو كان الحارس عامًا على تطبيق HTTP.
5. `close` يشغّل `onModuleDestroy` و `onApplicationShutdown`. بلاها قد تبقى مقابض مفتوحة وتعلق العملية.

## مثال عملي

مهمة ليلية في رف:

```ts
const job = app
  .select(BillingModule)
  .get(InvoiceExportJob, { strict: true });
await job.exportOrganization('org_nile');
```

القفل `@OnOneInstance` من فصل الاعتمادية يبقى لازمًا إذا شغّلت أكثر من آلة نفس السكربت. السياق المستقل لا يوحّد العمليات. مرّروا معرف المنظمة في سطر الأمر بعد التحقق، لا من ملف يرفعه عميل.

## أخطاء شائعة

- `listen` داخل سكربت التصدير.
- افتراض أن الحارس العام فحص المنظمة.
- `ConfigModule.register` مرتين بكائنين مختلفين ثم `select` على الثاني.
- نسيان `close` فيبقى عامل Kubernetes في حالة Running.

## متى نستخدمه؟

سكربت، مهمة تنتهي، عامل بلا HTTP، أو دالة serverless بلا مسارات كما في فصل المنصة الاختياري.

## متى لا نستخدمه؟

لواجهة رف التي تخدم المتصفح. تلك `NestFactory.create` ثم `listen`. ولا بدل اختبار وحدة يعزل مزودًا بـ `Test.createTestingModule`.

## مقارنة

| المصنع | الشبكة | الحراس على التابع |
| --- | --- | --- |
| `create` + `listen` | HTTP | على المسار |
| `createApplicationContext` | لا | لا |
| `create` + `init` بلا `listen` | لا HTTP، والحاوية كاملة | حسب ما تستدعون |

## تمارين

### تمرين سهل

أي استدعاء يغلق السكربت بشكل مرتب؟

<details>
<summary>الحل</summary>

`await app.close()` بعد انتهاء العمل. يطلق خطافات الإيقاف.

</details>

### تمرين متوسط

لماذا `app.get(controller)` ثم استدعاء التابع لا يطبّق الأنبوب؟

<details>
<summary>الحل</summary>

الأنابيب والوسيط والاعتراضات مربوطة بمعالجة الطلب. السياق المستقل لا يملك مستمعًا. التابع يُستدعى كدالة على الكائن.

</details>

### تمرين متقدم

`select(ConfigModule.register({ folder: './config' }))` لا يجد الخدمة رغم أن `AppModule` يستورد الإعداد. ما الشرط؟

<details>
<summary>الحل</summary>

مرّروا نفس مرجع الوحدة الديناميكية المصدَّر والمستورَد في `AppModule`. استدعاء `register` من جديد كائن آخر فلا يطابق الرسم.

</details>

## أسئلة مراجعة

- أي مصنع؟
- ماذا يفعل `strict: true`؟
- هل يوجد منفذ؟
- متى `close`؟

## الخلاصة

`createApplicationContext` يشغّل حاوية رف لمهمة بلا شبكة. المنظمة وسيط صريح. `close` ينهي العملية. الحارس لا يعمل لأنكم لم تمروا بمسار.

## ماذا بعد؟

إن صار للسكربت أوامر وخيارات، `nest-commander` يغلّف هذا النمط.

## المصادر الرسمية

- NestJS — Standalone applications: https://docs.nestjs.com/standalone-applications
