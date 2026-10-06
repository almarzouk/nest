# Consumers

## الهدف من هذا الدرس

تنفيذ مهمة `resize` في صنف يمتد `WorkerHost`، بقراءة `bookId` من الحمولة وإعادة جلب الصف داخل `process`.

## المشكلة التي يحلها

الطابور يتراكم إن لم يوجد معالج في `providers`. معالج يثق بجسم المهمة كأنه صف كامل يعمل على عنوان قديم أو على كائن لم يُسلسَل كما ظننت. المستهلك يعيد التحميل من Prisma ثم يقص.

## الفكرة ببساطة

`@Processor(coverQueueName)` على صنف يمتد `WorkerHost` من `@nestjs/bullmq`. تكتب `process(job)`. `job.data` من نوع `ResizePayload`. أي رمي داخل `process` فشل مهمة ستُعاد حسب السياسة. الصنف في `providers` مع `registerQueue` بالاسم نفسه.

## التعريف التقني

```ts
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { coverQueueName } from './queue-names.js';
import type { ResizePayload } from './covers.producer.js';
import { CoversService } from './covers.service.js';

@Processor(coverQueueName)
export class CoversConsumer extends WorkerHost {
  constructor(private readonly covers: CoversService) {
    super();
  }

  async process(job: Job<ResizePayload>): Promise<void> {
    if (job.name !== 'resize') return;
    await this.covers.resize(job.data.bookId, job.data.coverKey);
  }
}
```

`CoversService.resize` تفتح الصف بالمعرف. إن لم يوجد الكتاب، تخرج بهدوء أو ترمي حسب ما إن كان الغياب مؤقتًا. إن كان `coverKey` في القاعدة مختلفًا عن حمولة المهمة، تتوقف لأن تحديثًا أحدث تولى الغلاف.

لا تزين الدالة بـ `@Process`. تلك زينة `@nestjs/bull`.

## Mental Model

```text
Redis مهمة resize { bookId, coverKey }
  WorkerHost.process
    prisma.book.findUnique
    قص الملف إن تطابق المفتاح
    تحديث coverStatus = ready
```

## أبسط مثال

```ts
async resize(bookId: string, coverKey: string) {
  const book = await this.prisma.book.findUnique({ where: { id: bookId } });
  if (!book || book.coverKey !== coverKey) return;
  await this.images.writeThumbnail(coverKey);
  await this.prisma.book.update({
    where: { id: bookId },
    data: { coverStatus: 'ready' },
  });
}
```

`return` بلا رمي يعني نجاح المهمة حتى لا تُعاد إلى الأبد لكتاب حُذف أو استُبدل غلافه. الرمي يبقى لخطأ القرص أو قاعدة متقطعة حيث الإعادة مفيدة.

## شرح المثال

المقارنة مع `coverKey` تمنع قصًا قديمًا من كتابة `ready` فوق تحديث أحدث. `writeThumbnail` يقرأ من مسار بنته أنت من المفتاح المولَّد لا من اسم المستخدم. المستهلك لا يعرف `Request`.

## ماذا يحدث داخليًا؟

Nest تسجل العامل عند الإقلاع لأن الصنف مزوّد وعليه `@Processor`. BullMQ يحجز المهمة للعامل حتى لا يأخذها عامل ثانٍ معًا. `process` يعمل حتى يعود الوعد. النجاح ينقل المهمة إلى حالة مكتملة. الرمي يعيدها إلى فشل قابل لإعادة المحاولة. إن كان التطبيق بلا `@Processor` في شجرة الوحدات، المنتج ما زال يستطيع `add` والمهام تبقى في الانتظار. تحقق من طول الانتظار في التطوير.

عدة نسخ من التطبيق تعني عدة عمال على الطابور نفسه، وهذا مقصود لزيادة القص. لذلك `resize` يتحمل التنفيذ المتوازي لكتابين مختلفين، والكتاب الواحد محمي بـ `jobId` وبفحص `coverKey`.

## مثال عملي

التسجيل:

```ts
@Module({
  imports: [BullModule.registerQueue({ name: coverQueueName })],
  providers: [CoversProducer, CoversConsumer, CoversService],
})
export class CoversModule {}
```

`CoversService` قابلة للاختبار بلا Redis: استدعِ `resize('42', 'key')` على قاعدة اختبار. `process` غلاف رفيع.

## أخطاء شائعة

- `@Process()` من الحزمة القديمة على مسار BullMQ.
- استخدام `job.data` كصف كامل بلا `findUnique`.
- رمي `NotFoundException` لكتاب حُذف، فيتكرر الفشل حتى تنفد المحاولات.
- قص `coverKey` القديم وكتابة `ready` بلا مقارنة.

## متى نستخدمه؟

لكل اسم مهمة في الطابور: `resize` هنا، و `send` على طابور البريد بصنف معالج آخر حتى لا يختلط الفشل.

## متى لا نستخدمه؟

لمنطق يحتاجه الرد المتزامن. ولا تفتح عاملًا في عملية اختبار الوحدة التي لا تشغل Redis. اختبر `resize` مباشرة.

## مقارنة

| الرمي | مصير المهمة |
| --- | --- |
| نجاح الوعد | مكتملة |
| `return` بعد كتاب غائب | مكتملة بلا قص |
| خطأ قرص | فشل ثم إعادة حسب `attempts` |

## تمارين

### تمرين سهل

أي صنف يمتد المستهلك على المسار النشط؟

<details>
<summary>الحل</summary>

`WorkerHost`، والدالة `process`.

</details>

### تمرين متوسط

لماذا تعيد جلب الكتاب داخل `resize`؟

<details>
<summary>الحل</summary>

لأن الحمولة معرفات فقط، والحالة قد تتغير بين `add` والتنفيذ.

</details>

### تمرين متقدم

متى تخرج بهدوء بدل رمي `NotFoundException`؟

<details>
<summary>الحل</summary>

عندما يُحذف الكتاب أو يتبدل `coverKey`. الإعادة لن تصلحه. ارمِ فقط لما قد ينجح في المحاولة التالية مثل انقطاع التخزين.

</details>

## أسئلة مراجعة

- أين يُذكر اسم الطابور على المستهلك؟
- ماذا في `job.data`؟
- لماذا `process` رفيع و `resize` هي العمل؟

## الخلاصة

`WorkerHost.process` يقرأ `{ bookId, coverKey }` ويجلب الصف. النجاح يكمل المهمة. الغياب النهائي لا يُعاد رميه بلا نهاية.

## ماذا بعد؟

كم مرة يُعاد الفشل، وما الذي يجب أن يتحمله القص إن تكرر.

## المصادر الرسمية

- NestJS — Queues: https://docs.nestjs.com/application/queues
- BullMQ — NestJS: https://docs.bullmq.io/guide/nestjs
