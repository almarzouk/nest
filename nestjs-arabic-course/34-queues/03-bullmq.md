# BullMQ و @nestjs/bullmq

## الهدف من هذا الدرس

تسجيل الطابور النشط في Nest: اتصال في `forRoot`، واسم الطابور في `registerQueue` هو رمز الحقن.

## المشكلة التي يحلها

معرفة Redis لا تعطي `Queue` قابلة للحقن. المقالات التي تستعمل `@nestjs/bull` و `@Process` تصف Bull، وهو في وضع الصيانة حسب صفحة Nest. المسار الذي تكمله الدورة `@nestjs/bullmq` مع صنف معالج يمتد `WorkerHost`.

## الفكرة ببساطة

`BullModule.forRoot` مرة للاتصال الذي تشاركه الطوابير. `BullModule.registerQueue({ name })` لكل طابور. السلسلة `name` تُحقن بـ `@InjectQueue(name)` ويستخدمها `@Processor(name)`. الاسم نفسه على المنتج والمستهلك. اختلاف حرف يعني طابورًا صامتًا ومستهلكًا ينتظر غيره.

## التعريف التقني

```bash
npm i @nestjs/bullmq bullmq
```

```ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    BullModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        connection: {
          host: config.get<string>('QUEUE_REDIS_HOST') ?? 'localhost',
          port: Number(config.get<string>('QUEUE_REDIS_PORT') ?? 6379),
        },
      }),
    }),
  ],
})
export class AppModule {}
```

طوابير الميزة:

```ts
BullModule.registerQueue({ name: 'covers' });
```

التوثيق: `forRoot` يسجل إعداد BullMQ الذي تستخدمه الطوابير ما لم تُخصص. `name` رمز حقن وطريقة ربط المستهلك. لا تضع خيارات Bull القديمة `redis: { host, port }` على هذا المسار. تلك شكل الحزمة الأخرى.

`@nestjs/bull` تبقى موثّقة لمن يعتمد على Bull. لا تبدأ بها مشروع المتجر. درس أخير في الفصل يضع الفرق حتى لا تخلط الزينة.

## Mental Model

```text
forRoot connection
  registerQueue name: covers
    InjectQueue('covers') في المنتج
    @Processor('covers') في المستهلك
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { CoversProducer } from './covers.producer.js';

@Module({
  imports: [BullModule.registerQueue({ name: 'covers' })],
  providers: [CoversProducer],
  exports: [CoversProducer],
})
export class CoversModule {}
```

`forRoot` في الجذر أو في وحدة مستوردة مرة. `registerQueue` في الوحدة التي تنتج أو تستهلك. تكرار `forRoot` بإعدادين مختلفين يربك الاتصال الافتراضي.

## شرح المثال

الاسم `'covers'` ليس مسار HTTP. هو هوية القائمة في Redis ضمن بادئة المكتبة. المنتج الذي ستكتبه يحقن الطابور بهذا النص. المستهلك يجب أن يكرر النص نفسه. تصدير المنتج يسمح لـ `BooksModule` بإدراج المهمة بعد الحفظ بلا حقن `Queue` في المتحكم.

## ماذا يحدث داخليًا؟

عند الإقلاع تتصل المكتبة بـ Redis باستخدام `connection`. `registerQueue` ينشئ مزوّدًا مفتاحه الاسم. الحقن بالاسم يعيد فئة `Queue` من `bullmq`. إن كان Redis متوقفًا، يفشل الإقلاع أو أول `add` حسب لحظة الاتصال. لا تؤجل اكتشاف ذلك إلى أول زائر إن استطعت فحصًا عند البدء. العامل الفعلي يُربط عندما يوجد `@Processor` مسجّل في `providers`. تسجيل الطابور بلا معالج يعني مهامًا تتراكم بلا قص.

## مثال عملي

طابوران بأسماء ثابتة في ملف واحد حتى لا تُكتب السلسلة في ثلاثة مواضع بشكل مختلف:

```ts
export const coverQueueName = 'covers';
export const mailQueueName = 'mail';
```

`registerQueue({ name: coverQueueName })` و `@InjectQueue(coverQueueName)` و `@Processor(coverQueueName)` تستورد الثابت. البريد لا يمر على طابور الغلاف حتى لا يؤخر القص رسالة ولا العكس.

## أخطاء شائعة

- تثبيت `@nestjs/bull` ونسخ `@Process` في المسار الجديد.
- `forRoot({ redis: { host, port } })` على BullMQ.
- اسم `'Covers'` في المعالج و `'covers'` في التسجيل.
- `registerQueue` بلا `forRoot` في شجرة التطبيق.

## متى نستخدمه؟

لمهام المتجر الجديدة: غلاف، بريد، تصدير. كل قائمة اسم ثابت واتصال من الإعداد.

## متى لا نستخدمه؟

لجدولة «كل دقيقة» بلا حمولة عمل. ذلك `@nestjs/schedule` في الفصل التالي. BullMQ يستطيع التأخير، لكن ساعة التطبيق ليست أول سبب لإضافة طابور.

## مقارنة

| | `@nestjs/bullmq` | `@nestjs/bull` |
| --- | --- | --- |
| المكتبة | BullMQ النشطة | Bull في الصيانة |
| الاتصال | `connection` | `redis` |
| المستهلك | `WorkerHost.process` | `@Process` |
| مسار الدورة | نعم | معرفة بالفرق فقط |

## تمارين

### تمرين سهل

أي حقل في `forRoot` يحمل `host` و `port` على المسار النشط؟

<details>
<summary>الحل</summary>
`connection`.
</details>

### تمرين متوسط

ماذا يوحّد المنتج بالمستهلك؟

<details>
<summary>الحل</summary>
سلسلة `name` نفسها في `registerQueue` و `@InjectQueue` و `@Processor`.
</details>

### تمرين متقدم

لماذا الثابت `coverQueueName` أفضل من تكرار النص؟

<details>
<summary>الحل</summary>
لأن اختلاف حالة الأحرف يصنع طابورين. الثابت يجعل التسجيل والحقن والمعالج يستوردون قيمة واحدة.
</details>

## أسئلة مراجعة

- أين يُستدعى `forRoot` مقارنة بـ `registerQueue`؟
- ما رمز الحقن؟
- أي حزمة لا تبدأ بها مشروعًا جديدًا؟

## الخلاصة

`BullModule.forRoot` يتصل و `registerQueue` يسمي الطابور. الاسم رمز الحقن. المسار النشط `@nestjs/bullmq`.

## ماذا بعد؟

المنتج: `queue.add` بعد حفظ الكتاب بحمولة `{ bookId }`.

## المصادر الرسمية

- NestJS — Queues: https://docs.nestjs.com/application/queues
