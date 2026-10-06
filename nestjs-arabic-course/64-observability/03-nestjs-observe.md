# NestJS Observe و @nestjs/observe

## الهدف من هذا الدرس

تشغيل المراقبة الرسمية لرف بحزمة `@nestjs/observe`: خطاف على دورة Nest لا عميل OpenTelemetry تجمّعونه أنتم، مع معرفة أنه اختياري عند `nest new`.

## المشكلة التي يحلها

تريدون p95 لكل مسار وأسماء `BooksController.publish` في اللوحة، بلا أسبوع مجمع وexporter. في الوقت نفسه لا تريدون أن يُفرض ذلك على كل مشروع تعليمي. أمر `nest new` يسأل أو يقبل علمًا صريحًا.

## الفكرة ببساطة

`createObserveModule()` يعيد `ObserveModule` و `ObserveInstrument`. تستوردون `ObserveModule.forRoot` في الجذر وتمررون `instrument: ObserveInstrument` إلى `NestFactory.create` حتى يلحق SDK قبل الطلبات. الحزمة ترسل إلى NestJS Observe. هي ليست بديل OpenTelemetry لمن يجمع عدة لغات. عند إنشاء المشروع، `--observe` يفعّلها و `--no-observe` يتخطاها.

## التعريف التقني

من ملاحظات Nest 12: `nest new` و `nest upgrade` يستطيعان توصيل الحزمة بـ `--observe`. العلم `--no-observe` يجاوب السؤال بالنفي عندما لا تريدون أن يُضاف SDK. التوثيق: الحزمة تُعلّق على دورة الطلب عبر خيار `instrument` لا عبر لف خادم HTTP كعميل عام. `ObserveModule.forRoot` يحتاج على الأقل معرّف خدمة (ومفاتيح التطبيق حسب الصفحة الحالية). مع Fastify، المحول الوسيط الثاني والخيار `instrument` في الثالث، مثل `rawBody`. الإصدار المطلوب من النواة يتضمن خيار `instrument` (الوثائق تذكر 11.1.4 كحد أدنى، ورف على Nest 12 فوق ذلك). GraphQL يحتاج نسخة graphql متوافقة إن وُجد. لا تحذفوا `/health`؛ الموازن ما زال يحتاجه وصفحة المراقبة تقول ذلك.

## Mental Model

```text
nest new --observe     → ملفات جاهزة
أو --no-observe        → بلا SDK
createObserveModule → forRoot + instrument في create
```

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule, ObserveInstrument } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { instrument: ObserveInstrument });
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

## شرح المثال

- `ObserveInstrument` من نفس `createObserveModule()` الذي صدّر الوحدة، لا من استدعاء ثانٍ.
- Express: الخيار ثاني وسيط. لا تضعوا المحول هنا.
- من غير `instrument` قد تُستورد الوحدة ولا تُلقَّن الدورة.

## ماذا يحدث داخليًا؟

1. `createObserveModule()` يبني الزوج المرتبط بإعداد واحد.
2. `forRoot` يسجّل الإرسال بمعرّف الخدمة.
3. `instrument` يلتقط المتحكمات والمهام قبل الاستماع.
4. الطلبات تُرسل بأسماء أصنافكم. استعلامات SQL تظهر بشكل الجملة لا بالقيم حسب وصف المنتج.
5. بيئة متعددة اللغات تبقى على OpenTelemetry إن كان المجمع المشترك قرار المنظمة. يمكن فهم الاثنين، لكن لا تتوقعوا أن لوحة Observe تستبدل `traceparent` لكل Python تلقائيًا.

## مثال عملي

```ts
import { Module } from '@nestjs/common';
import { createObserveModule } from '@nestjs/observe';

export const { ObserveModule, ObserveInstrument } = createObserveModule();

@Module({
  imports: [
    ObserveModule.forRoot({
      serviceId: 'raf-api',
      appKey: process.env.OBSERVE_APP_KEY!,
      appSecret: process.env.OBSERVE_APP_SECRET!,
    }),
  ],
})
export class AppModule {}
```

أسماء `appKey` و `appSecret` اتبعوا صفحة SDK إن تغيّر التوقيع؛ الفكرة أن السر في البيئة لا في المستودع. `serviceId` يفرّق API عن العامل.

## أخطاء شائعة

- نسيان `instrument` والاكتفاء بالوحدة.
- وضع `instrument` مكان `FastifyAdapter`.
- ادّعاء أن الحزمة تلغي OpenTelemetry في شركة متعددة اللغات.
- إيقاف health لأن اللوحة تراقب. الموازن لا يفتح لوحة Observe.

## متى نستخدمه؟

عندما تريدون مراقبة Nest بأسماء الإطار بسرعة، وتقبلون خدمة NestJS Observe.

## متى لا نستخدمه؟

عندما يكون مجمع OpenTelemetry معيار المنظمة لكل اللغات. وعندها `--no-observe` على `nest new` يبقي المشروع بلا SDK.

## مقارنة

| | OpenTelemetry مجمعكم | `@nestjs/observe` |
| --- | --- | --- |
| الأسماء | حسب التلقين | متحكمات ومزوّدات Nest |
| التشغيل | تصدير ومجمّع | مفاتيح وخيار `instrument` |
| إنشاء المشروع | لا يُضاف تلقائيًا إلا بقرار | `--observe` / `--no-observe` |
| متعدد اللغات | مناسب | ليس بديل المعيار |

## تمارين

### تمرين سهل

أي علم يمنع إضافة الحزمة عند `nest new`؟

<details>
<summary>الحل</summary>

`--no-observe`. التفعيل الصريح `--observe`.

</details>

### تمرين متوسط

الوحدة مستوردة ولا تظهر طلبات. ماذا ينقص `bootstrap`؟

<details>
<summary>الحل</summary>

تمرير `instrument: ObserveInstrument` إلى `NestFactory.create`. على Fastify يكون ذلك الوسيط الثالث بعد المحول.

</details>

### تمرين متقدم

لماذا تبقون Terminus مع Observe؟

<details>
<summary>الحل</summary>

لأن الموازن يضرب مسار صحة ولا يقرأ لوحة المراقبة. Observe يفسّر البطء والأخطاء. الغياب عن اللوحة لا يعني أن الحاوية يجب أن تُسحب من الخدمة فورًا إن كان الفحص المحلي أخضر، والعكس صحيح أيضًا: الفحص الأخضر لا يشرح p95.

</details>

## أسئلة مراجعة

- ماذا يعيد `createObserveModule`؟
- أين يُمرَّر `ObserveInstrument`؟
- كيف تتجاوزون السؤال عند إنشاء المشروع؟
- لماذا ليست الحزمة بديلًا إلزاميًا عن OpenTelemetry؟

## الخلاصة

`@nestjs/observe` اختياري عبر `--observe` و `--no-observe`، ويُربط بـ `forRoot` وخيار `instrument` قبل الطلبات. الأسماء من Nest. المعيار متعدد اللغات يبقى OpenTelemetry. مسار الصحة يبقى.

## ماذا بعد؟

سطر السجل بلا مفتاح لا يُجمع مع الأثر. الدرس التالي معرّف يربط سجلات الطلب الواحد.

## المصادر الرسمية

- NestJS — Observability SDK: https://docs.nestjs.com/observability/sdk
- NestJS — CLI: https://docs.nestjs.com/cli/overview
