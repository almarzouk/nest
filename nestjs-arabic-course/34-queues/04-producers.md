# Producers

## الهدف من هذا الدرس

إدراج مهمة غلاف بعد حفظ الكتاب، بحقن الطابور الذي سجّلته، وبحمولة معرفات فقط.

## المشكلة التي يحلها

المتحكم الذي ينادي `Queue` من `bullmq` مباشرة ينشر اسم الطابور وشكل الحمولة في طبقة HTTP. وإن أدرجت المهمة قبل `insert` ثم فشل الحفظ، يقص العامل كتابًا غير موجود. المنتج خدمة صغيرة تُستدعى بعد نجاح القاعدة.

## الفكرة ببساطة

`@InjectQueue(coverQueueName)` يحقن `Queue`. `add(jobName, payload)` يكتب المهمة. `jobName` مثل `resize`. الحمولة `{ bookId, coverKey }`. لا كائن Prisma ولا `Buffer`. المتحكم يحفظ ثم يستدعي المنتج ثم يرد `201`.

## التعريف التقني

```ts
import { Inject, Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { coverQueueName } from './queue-names.js';

export interface ResizePayload {
  bookId: string;
  coverKey: string;
}

@Injectable()
export class CoversProducer {
  constructor(@InjectQueue(coverQueueName) private readonly covers: Queue) {}

  enqueueResize(payload: ResizePayload) {
    return this.covers.add('resize', payload, {
      jobId: `resize:${payload.bookId}`,
    });
  }
}
```

`jobId` الثابت يجعل إعادة الإدراج للكتاب نفسه لا تنشئ مهمتين متوازيتين بالاسم نفسه بينما الأولى ما زالت قائمة، حسب سلوك BullMQ في تجاهل التكرار. هذا يساعد إن أعاد الزائر `POST` التحديث. لا تعتمد عليه بدل جعل القص نفسه آمنًا للتكرار.

`add` يعيد وعدًا بالمهمة المخزّنة. لا تنتظر `waitUntilFinished` في مسار الزائر.

## Mental Model

```text
BooksService.create
  prisma.book.create
  covers.enqueueResize({ bookId, coverKey })
  return { id, coverStatus: 'pending' }
```

## أبسط مثال

```ts
async create(input: { title: string; coverKey: string }) {
  const book = await this.prisma.book.create({
    data: { title: input.title, coverKey: input.coverKey, coverStatus: 'pending' },
  });
  await this.covers.enqueueResize({ bookId: book.id, coverKey: book.coverKey });
  return book;
}
```

`coverKey` أتى من الرفع، وليس من كائن الملف. `coverStatus` عمود حتى تستطيع الواجهة الاستعلام إن تأخر العامل.

## شرح المثال

الترتيب: صف ثم مهمة. إن رمى `add`، الكتاب موجود والقص غير مجدول. السجل يجب أن يظهر ذلك، ومسار لاحق يستطيع إعادة `enqueueResize` لأن `jobId` مرتبط بالكتاب. لا تحذف الكتاب لأن Redis تعطل لحظة. لا تمرر `book` الناتج من Prisma إلى `add`. الحقلان نصان.

## ماذا يحدث داخليًا؟

`InjectQueue` يحل المزود الذي صنعه `registerQueue` بالاسم. `add` يسلسل JSON إلى Redis ويعيد معرف المهمة. العامل الذي ينام على الطابور يستيقظ على نسخة أخرى أو على العملية نفسها إن كان `@Processor` فيها. المنتج لا يعرف أي عملية ستنفذ. لذلك ممنوع تمرير دالة أو اتصال قاعدة داخل الحمولة.

## مثال عملي

تحديث الغلاف يعيد الإدراج:

```ts
await this.prisma.book.update({
  where: { id: bookId },
  data: { coverKey, coverStatus: 'pending' },
});
await this.covers.enqueueResize({ bookId, coverKey });
```

إن كانت المهمة القديمة ما زالت تعمل على `coverKey` السابق، القص يجب أن يكتب الناتج فقط إن كان المفتاح في القاعدة ما زال هو مفتاح المهمة. ذلك فحص داخل المستهلك، لأن المنتج لا يلغي العمل الجاري بمجرد `add` جديد إلا إذا صممت إلغاءً صريحًا.

## أخطاء شائعة

- `add('resize', book)` و `book` نموذج Prisma.
- `job.finished()` قبل `return` في المتحكم.
- إدراج المهمة ثم الحفظ.
- حقن `Queue` في المتحكم ونسخ اسم الطابور هناك.

## متى نستخدمه؟

بعد كل حفظ يجعل عملًا مؤجلًا مستحقًا: غلاف جديد أو بريد بعد انتقال الحالة إلى منشور.

## متى لا نستخدمه؟

قبل التحقق من DTO. العامل ليس مكان `class-validator`. ولا لكل `GET`. القراءة مخبأ أو استعلام، لا طابور.

## مقارنة

| الاستدعاء | ماذا ينتظر الزائر |
| --- | --- |
| `await enqueueResize` | تأكيد Redis فقط |
| `await job.waitUntilFinished()` | انتهاء القص |
| قص متزامن في الخدمة | انتهاء القص أيضًا، بلا إعادة محاولة من الطابور |

## تمارين

### تمرين سهل

أي زينة تحقن الطابور باسمه؟

<details>
<summary>الحل</summary>

`@InjectQueue(coverQueueName)` من `@nestjs/bullmq`.

</details>

### تمرين متوسط

لماذا `jobId` المشتق من `bookId` مفيد عند إعادة الحفظ؟

<details>
<summary>الحل</summary>

لأنه يحد من مهمتين نشطتين بالهوية نفسها للكتاب، فيقل التوازي المكرر. القص ما زال يجب أن يتحمل التكرار.

</details>

### تمرين متقدم

فشلت `add` بعد `create`. ماذا يبقى، وماذا لا تفعله؟

<details>
<summary>الحل</summary>

الكتاب في القاعدة بلا مهمة. لا تحذفه تلقائيًا. سجّل الفشل وأعد الإدراج. الزائر يمكن أن يرى `pending` مع علم أن الجدولة ستُعاد.

</details>

## أسئلة مراجعة

- ما حقول `ResizePayload`؟
- لماذا لا يُمرَّر نموذج Prisma؟
- متى يُستدعى المنتج نسبة إلى `insert`؟

## الخلاصة

المنتج يحقن الطابور بالاسم ويضيف `{ bookId, coverKey }` بعد الحفظ. الرد لا ينتظر القص.

## ماذا بعد؟

المستهلك الذي يقرأ هذه الحمولة وينفذ `process`.

## المصادر الرسمية

- NestJS — Queues: https://docs.nestjs.com/application/queues
