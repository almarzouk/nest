# Correlation IDs

## الهدف من هذا الدرس

إرفاق معرف واحد بكل أسطر الطلب نفسه، وتمريره إلى حمولة المهمة حتى يظهر في العامل بعد أن أغلق الزائر الاتصال.

## المشكلة التي يحلها

سطر `POST /books` وسطر `isbn timed out` وسطر `resize failed` متباعدة في المجمع. بلا مفتاح مشترك لا تعرف أنها طلب الزائر نفسه. معرف التتبع يُولَّد عند الدخول ويُقرأ حيث يُكتب السجل، ويُنسخ إلى `{ bookId, requestId }` إن كان يجب أن يعيش بعد الرد.

## الفكرة ببساطة

وسيط HTTP يضع `requestId` من ترويسة `x-request-id` إن كانت آمنة، أو من `randomUUID()`. يخزنه في `AsyncLocalStorage` للعملية غير المتزامنة على الطلب. المسجّل يقرأ المخزن ويضيف الحقل. عند `queue.add` انسخ المعرف إلى الحمولة لأن العامل سياقًا آخر لا يرى التخزين المحلي للطلب.

## التعريف التقني

```ts
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

export const requestContext = new AsyncLocalStorage<{ requestId: string }>();

export function requestIdMiddleware(req: { headers: Record<string, string | undefined> }, _res: unknown, next: () => void) {
  const incoming = req.headers['x-request-id'];
  const requestId = incoming && /^[\w-]{8,64}$/.test(incoming) ? incoming : randomUUID();
  requestContext.run({ requestId }, next);
}
```

الشرط على الترويسة يمنع سطر سجل فيه مسافة أو حقن سطر جديد. لا تثق بطول غير محدود.

في `write` داخل المسجّل:

```ts
const requestId = requestContext.getStore()?.requestId ?? null;
```

أضف الحقل إلى JSON. إن كان `null` فالسطر من مؤقت أو إقلاع لا من طلب، وهذا صادق.

نسخ إلى الطابور:

```ts
await this.covers.add('resize', {
  bookId: book.id,
  coverKey: book.coverKey,
  requestId: requestContext.getStore()?.requestId,
});
```

العامل يسجّل `job.data.requestId` مع `bookId`. لا تعتمد على `AsyncLocalStorage` داخل `process` إلا إذا أنشأت سياقًا جديدًا من الحمولة في أول `process`. الأبسط قراءة الحقل من `job.data`.

## Mental Model

```text
وسيط -> requestId في AsyncLocalStorage
  سطر HTTP
  سطر ISBN
  add { requestId }
    عامل يقرأ job.data.requestId
```

## أبسط مثال

سطر `request` وسطر `published` يحملان `requestId` نفسه. البحث في المجمع عن القيمة يعيد السلسلة.

## شرح المثال

الوسيط يجب أن يغلّف `next()` داخل `run` حتى تبقى وعود المتحكم داخل السياق. استدعاء `run` ثم `next()` خارج الدالة يفقد المعرف عند أول `await`. هذا هو الخطأ الشائع مع `AsyncLocalStorage`.

معرف من الموازن مقبول إن طابق النمط. وإلا ولّد واحدًا حتى لا يرفض الطلب بسبب ترويسة قبيحة، ولا تنسخ القبيحة إلى السجل.

## ماذا يحدث داخليًا؟

`AsyncLocalStorage` يتتبع استمرار `async` في Node للطلب. `emitAsync` والمستمع الذي يُستدعى من مسار الطلب يرث السياق إن بقي على السلسلة نفسها. `setInterval` في `@Interval` لا يرث طلبًا، فـ `requestId` هناك `null`. مهمة BullMQ تُنفَّذ لاحقًا على مكدس العامل، لذلك النسخ الصريح في الحمولة هو الجسر. لا تضع المعرف بدل `jobId`. `jobId` ما زال `resize:${bookId}` لمنع التكرار. `requestId` للتتبع فقط وقد يتعدد مع إعادة الحفظ.

## مثال عملي

حمولة القص `{ bookId, coverKey, requestId? }`. العامل يسجّل `job.data.requestId ?? null` مع `bookId`. منتج قديم بلا الحقل لا يُفشل المهمة. المعرف للتتبع، والصف يُجلب بـ `bookId`.

## أخطاء شائعة

- `next()` خارج `requestContext.run`.
- نسخ ترويسة خام فيها حرف سطر.
- توقع المعرف داخل `@Cron` بلا طلب.
- استخدام `requestId` كـ `jobId` فيضيع دمج مهام الكتاب الواحد.

## متى نستخدمه؟

لكل طلب وارد في الإنتاج، ومع المهام التي تنشأ منه.

## متى لا نستخدمه؟

كبديل عن `bookId` في الحمولة. المعرف يربط السجلات ولا يجلب الصف. ولا تمرره إلى الزائر كسر.

## مقارنة

| السياق | من أين يأتي المعرف |
| --- | --- |
| طلب HTTP | الوسيط و `AsyncLocalStorage` |
| مستمع على السلسلة نفسها | التخزين نفسه |
| عامل الطابور | `job.data.requestId` |
| `@Cron` | لا طلب، الحقل `null` |

## تمارين

### تمرين سهل

ماذا تولّد إن غابت الترويسة أو كانت غير آمنة؟

<details>
<summary>الحل</summary>

`randomUUID()` ولا تُنسخ الترويسة القبيحة إلى السجل.

</details>

### تمرين متوسط

لماذا لا يرى `process` التخزين المحلي للطلب تلقائيًا؟

<details>
<summary>الحل</summary>

لأن العامل يعمل لاحقًا وربما في عملية أخرى. انسخ `requestId` إلى الحمولة.

</details>

### تمرين متقدم

لماذا لا يكون `requestId` هو `jobId` لمهمة القص؟

<details>
<summary>الحل</summary>

لأن `jobId` يمنع تكرار قص الكتاب (`resize:${bookId}`). معرف الطلب يتغير مع كل `POST` ولو للكتاب نفسه.

</details>

## أسئلة مراجعة

- أين يُفحص شكل `x-request-id`؟
- أي حقل في JSON يربط السطور؟
- ماذا يُسجَّل في نبضة بلا طلب؟

## الخلاصة

معرف الطلب يولد عند الدخول ويسافر عبر `AsyncLocalStorage` ثم يُنسخ إلى حمولة المهمة. هو للسجلات، و `bookId` يبقى مفتاح العمل.

## ماذا بعد؟

اختبار الوحدة والتكامل والرحلة، حتى يثبت السجل والخدمة من غير تشغيل كل Redis في كل ملف.

## المصادر الرسمية

- NestJS — Logger: https://docs.nestjs.com/techniques/logger
- Node.js — `AsyncLocalStorage`: https://nodejs.org/api/async_context.html
