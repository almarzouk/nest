# NestFactory

## الهدف من هذا الدرس

معرفة أن المصنع هو الباب الوحيد الذي تبني منه التطبيق في هذه الدورة، وما الذي يعيده `create`.

## المشكلة التي يحلها

تجميع الحاوية ومحوّل HTTP والوحدات يدويًا يكرر ما يفعله الإطار ويفوّت ترتيب الإقلاع. المصنع يوحّد هذه الخطوات.

## الفكرة ببساطة

`NestFactory` من `@nestjs/core` فيه دوال إنشاء التطبيق. `create` ترجع تطبيق HTTP. دوال أخرى لاحقًا تنشئ سياقًا بلا خادم عندما نحتاج Standalone.

## التعريف التقني

التوثيق: `create` يعيد كائنًا يطبّق `INestApplication`. تمرير نوع عام مثل `NestExpressApplication` اختياري ويظهر دوال المنصة على الكائن. المعامل الثاني للمحوّل يختار Fastify بدل Express. الافتراضي Express عبر `@nestjs/platform-express`.

## Mental Model

```text
NestFactory.create(RootModule, adapter?)
        |
        +-- يقرأ الوحدة الجذر
        +-- يبني الحاوية
        +-- يركب محوّل HTTP
        |
        v
INestApplication
```

## أبسط مثال

```ts
const app = await NestFactory.create(AppModule);
```

## شرح المثال

- `AppModule` الجذر لا كائن Express.
- الناتج `app` ليس دالة معالج طلب. هو واجهة الإطار: `listen` و `useGlobalPipes` وما يشبههما في الفصول القادمة.
- من غير `await` تحصل على Promise لا على التطبيق.

## ماذا يحدث داخليًا؟

المصنع ينشئ حاوية، يحمّل الوحدات المستوردة بشكل متحمّس (eager) ما لم نستخدم تحميلًا كسولًا لاحقًا، يسجّل المزوّدات، ينشئها حسب الرسم، ويسجّل المسارات على المحوّل. بعد عودة `create` المسارات موجودة حتى لو لم يُفتح المنفذ بعد. هذا يفيد الاختبار: يمكن طلب الخادم الداخلي من غير `listen` على منفذ عام، كما سترى مع Supertest.

## مثال عملي

الوصول إلى API منصة Express فقط عند الحاجة:

```ts
import { NestFactory } from '@nestjs/core';
import type { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module.js';

const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

القوس الزاوي نوع فقط. هو لا يبدّل المنصة. تبديل Fastify يكون بتمرير `new FastifyAdapter()` كما في فصل المقدمة. لا تفعل ذلك في هذا الدرس.

## أخطاء شائعة

- إنشاء `express()` إضافي بجانب المصنع.
- اعتبار نوع `NestExpressApplication` تبديلًا للمنصة.
- استدعاء `create` أكثر من مرة في العملية نفسها للحصول على "نسختين" من غير قصد. كل استدعاء تطبيق مستقل.

## متى نستخدم create؟

لكل تطبيق HTTP في المسار الأساسي.

## متى نستخدم مصنعًا بلا HTTP؟

عندما تبني مهمة أو أمرًا لا يستمع إلى طلبات. `NestFactory.createApplicationContext` مسار Standalone في التوثيق. نؤجله حتى لا نخلط الإقلاع.

## مقارنة

| استدعاء | الناتج |
| --- | --- |
| `create(AppModule)` | تطبيق HTTP على Express |
| `create(AppModule, new FastifyAdapter())` | تطبيق HTTP على Fastify |
| سياق Standalone | وحدات وحاوية بلا منفذ |

## تمارين

### تمرين سهل

من أي حزمة يُستورد `NestFactory`؟

<details><summary>الحل</summary>

`@nestjs/core`.

</details>

### تمرين متوسط

ماذا يملك `app` بعد `create` وقبل `listen`؟

<details><summary>الحل</summary>

تطبيقًا مبنيًا: وحدات ومزوّدات ومسارات مربوطة على المحوّل. المنفذ لم يُفتح بعد.

</details>

### تمرين متقدم

لماذا اختبار قد يستدعي `create` ولا يستدعي `listen`؟

<details><summary>الحل</summary>

لأن المسارات مسجّلة على خادم المنصة الداخلي بعد البناء. أداة مثل Supertest تستطيع إرسال طلب إلى هذا الخادم من غير فتح منفذ على الجهاز. `listen` لازم عندما تريد عميلًا خارج العملية.

</details>

## أسئلة مراجعة

- ماذا يعيد `create`؟
- ماذا لا يفعل النوع العام `NestExpressApplication`؟
- متى تكون المسارات جاهزة بالنسبة إلى `listen`؟

## الخلاصة

`NestFactory.create` يبني تطبيق HTTP من الوحدة الجذر. المنصة افتراضيًا Express. الاستماع خطوة لاحقة.

## ماذا بعد؟

الوحدة التي تُمرَّر إلى `create` هي الجذر. نسمي دورها.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — Modules: https://docs.nestjs.com/modules
- NestJS — Standalone applications: https://docs.nestjs.com/standalone-applications
