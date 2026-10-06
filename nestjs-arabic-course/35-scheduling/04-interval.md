# Interval

## الهدف من هذا الدرس

تشغيل دالة كل مدة ثابتة بالميلي ثانية، من لحظة تسجيل المؤقت لا من خانة على الساعة.

## المشكلة التي يحلها

قياس طول طابور `covers` كل ثلاثين ثانية لا يحتاج نمط cron. كتابة `*/30 * * * * *` صحيحة لكنها أصعب قراءة لنبضة تشخيص. `@Interval` يصرّح بالمدة directly. هو لا يستبدل التقرير اليومي.

## الفكرة ببساطة

`@Interval(30_000)` يستدعي الدالة كل 30 ثانية. الرقم بالميلي ثانية كبقية المدد في هذه الفصول. الدالة تقرأ مقياسًا أو تضع علمًا. إن احتاجت قاعدة، فالاستعلام رخيص. العمل الثقيل يُدرج إلى طابور.

## التعريف التقني

```ts
import { Injectable } from '@nestjs/common';
import { Interval } from '@nestjs/schedule';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { coverQueueName } from '../covers/queue-names.js';

@Injectable()
export class QueueWatchService {
  constructor(@InjectQueue(coverQueueName) private readonly covers: Queue) {}

  @Interval(30_000)
  async logWaiting() {
    const waiting = await this.covers.getJobCounts('waiting');
    if (waiting.waiting > 100) {
      this.logger.warn(`covers waiting ${waiting.waiting}`);
    }
  }
}
```

`getJobCounts` قراءة حالة لا تنفيذ قص. التحذير عبر `Logger` ستفصله في فصل التسجيل. هنا المهم أن النبضة لا تنادي `resize`.

اسم اختياري يميّز المؤقت إن أردت إيقافه لاحقًا عبر `SchedulerRegistry`: `@Interval('covers-waiting', 30_000)`.

## Mental Model

```text
إقلاع
  30s -> logWaiting
  30s -> logWaiting
  ...
```

إعادة التشغيل تعيد العد من الصفر. لا التقاط لنبضة فاتت.

## أبسط مثال

```ts
@Interval(30_000)
async logWaiting() {
  const counts = await this.covers.getJobCounts('waiting');
  if (counts.waiting > 100) {
    this.logger.warn(`covers waiting ${counts.waiting}`);
  }
}
```

حد 100 رقم تشغيلي تضعه في الإعداد إن تعدد البيئات. لا تقص المهام داخل هذا الفرع.

## شرح المثال

ثلاثون ألف ميلي ثانية بين بداية الاستدعاءات حسب سلوك المكتبة. إن استمر `getJobCounts` نادرًا أطول من المدة، قد تتداخل النبضات. القراءة القصيرة لا تحتاج قفلًا. إن تحولت الدالة إلى مسح جدول كبير، أعدها إلى `@Cron` ليلي أو إلى مهمة طابور.

## ماذا يحدث داخليًا؟

`setInterval` في العملية يجدول الدالة. لا Redis للجدولة نفسها، لكن `getJobCounts` يسأل Redis الخاص بـ BullMQ. سقوط Redis يرمي داخل النبضة. التقط خطأ الاتصال وسجّله مرة، ولا تبنِ حلقة إعادة داخل `@Interval` لأن النبضة التالية ستحاول أصلًا. نسختان تعني تحذيرين كل 30 ثانية. للقياس هذا مقبول. لإجراء تغيير في القاعدة من النبضة راجع قفل الفصل السابق.

## مثال عملي

نبضة تتفقد كتب `pending` أقدم من خمس دقائق وتعيد `enqueueResize` بمعرّف المهمة نفسه:

```ts
@Interval(60_000)
async requeueStuck() {
  const stuck = await this.books.pendingOlderThan(5 * 60_000);
  for (const book of stuck) {
    await this.producer.enqueueResize({ bookId: book.id, coverKey: book.coverKey });
  }
}
```

هذا تعويض عن نافذة «حُفظ الصف وفشل `add`»، لا بديل عن `attempts` لمهمة وصلت الطابور وفشل قصها. `jobId` يمنع تكديس النسخ. إن كانت القائمة ضخمة، أدرجها كمهمة واحدة `requeue-stuck` بدل الحلقة في النبضة.

## أخطاء شائعة

- `@Interval(30)` باعتقاد أنها ثوانٍ، فتعمل كل 30 ميلي ثانية.
- قص أو بريد جماعي داخل النبضة.
- استبدال `@Cron('0 0 1 * * *')` بفترة 24 ساعة من وقت النشر.
- نسيان أن كل نسخة تقيس وتكتب إن كانت النبضة تغيّر بيانات.

## متى نستخدمه؟

لتباعد ثابت: مقياس، أو التقاط مهام عالقة، أو تنظيف لا يهتم بالدقيقة على الساعة.

## متى لا نستخدمه؟

لموعد تقويمي ومنطقة زمنية. ولا كمهلة طلب HTTP.

## مقارنة

| الزينة | مرجع الزمن |
| --- | --- |
| `@Interval(30_000)` | تباعد 30 ثانية في العملية |
| `@Cron('0 15 * * * *')` | الدقيقة 15 |
| `timeout` في Axios | سقف طلب واحد |

## تمارين

### تمرين سهل

ماذا تعني `30_000` في `@Interval`؟

<details>
<summary>الحل</summary>

30 ثانية، لأن الوحدة ميلي ثانية.

</details>

### تمرين متوسط

لماذا `requeueStuck` يستعمل `jobId` الموجود في المنتج؟

<details>
<summary>الحل</summary>

حتى لا تُضاف مهمة قص ثانية للكتاب نفسه إن كانت الأولى ما زالت قائمة.

</details>

### تمرين متقدم

متى تُخرج الحلقة من `@Interval` إلى مهمة طابور واحدة؟

<details>
<summary>الحل</summary>

عندما يكبر عدد الكتب العالقة ويطول عمل النبضة أو يتداخل. النبضة تدرج مهمة، والعامل يمشي على المعرفات.

</details>

## أسئلة مراجعة

- ما الفرق بين التباعد والتقويم؟
- ماذا تفعل النبضة إن سقط Redis؟
- أي أثر لا يكون داخل `logWaiting`؟

## الخلاصة

`@Interval` بالميلي ثانية لتباعد ثابت داخل العملية. اقرأ أو أعد إدراج معرفات. لا تنفّذ القص ولا تعتمد عليه لساعة الفجر.

## ماذا بعد؟

`@Timeout` لنبضة واحدة بعد الإقلاع، لا كل فترة.

## المصادر الرسمية

- NestJS — Task scheduling: https://docs.nestjs.com/techniques/task-scheduling
