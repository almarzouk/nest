# Server-Sent Events

## الهدف من هذا الدرس

دفع حالة تصدير أو معالجة غلاف إلى محرّر رف عبر `@Sse()` ودفق `text/event-stream`، وإغلاق الموارد حين يقطع المتصفح.

## المشكلة التي يحلها

المحرر يستعلم كل ثانية عن `GET /projects/:id`. أو اتصال يُغلق والـ `setInterval` يبقى. أو إعداد جلسة قاعدة أثناء `Promise` والعميل انصرف قبل الاشتراك فيُسرَّب المؤشر.

## الفكرة ببساطة

`@Sse()` من `@nestjs/common` يعيد `Observable<MessageEvent>` أو وعدًا به. المتصفح يستخدم `EventSource`. الحقول: `data` و `id` و `type` (يُرسل كحقل `event`) و `retry` و `comment`. `data` الكائن يُسلسل JSON. بلا `id` تضع Nest معرفًا متزايدًا. قطع العميل يلغي الاشتراك. `finalize` للتنظيف. `@SseSignal()` يعطي `AbortSignal` لعمر الاستجابة، بما فيه الانقطاع أثناء الإعداد.

## التعريف التقني

المسار يجب أن يعيد Observable. مثال الصفحة:

```ts
@Sse('events')
events(): Observable<MessageEvent> {
  return interval(1000).pipe(
    map(() => ({ data: { hello: 'world' } })),
  );
}
```

`interval` و `map` و `finalize` و `EMPTY` من `rxjs`. `MessageEvent` و `Sse` و `SseSignal` من `@nestjs/common`. الواجهة تسمح `data?: string | object`. تعليق `comment` سطر تبقيه المواصفة حيًا ويتجاهله `EventSource`. عند الإغلاق تستدعي Nest إلغاء الاشتراك. إن كان المعالج `async` وانقطع العميل قبل أن يُحَل الوعد، لا تُنشأ اشتراكًا في Observable المتأخر. افحصوا `signal.aborted` خلال الإعداد، أغلقوا المورد، وأعيدوا `EMPTY`. الإشارة تُجهض أيضًا عند اكتمال الدفق أو خطئه، فاجعلوا التنظيف قابلًا للتكرار. `@SseSignal()` على غير مسار `@Sse` يكون `undefined`. يعمل على Express و Fastify.

## Mental Model

```text
EventSource → GET يبقى مفتوحًا
كل next → كتلة تنتهي بسطرين فارغين
قطع أو اكتمال → إلغاء الاشتراك والإشارة
```

## أبسط مثال

```ts
import { MessageEvent, Sse } from '@nestjs/common';
import { finalize, interval, map, type Observable } from 'rxjs';

@Sse('projects/:id/activity')
activity(): Observable<MessageEvent> {
  return interval(2000).pipe(
    map(() => ({ type: 'activity', data: { status: 'processing' } })),
    finalize(() => this.releases.release()),
  );
}
```

العميل:

```js
const source = new EventSource('/projects/prj_season/activity');
source.addEventListener('activity', (event) => {
  console.log(JSON.parse(event.data));
});
```

بلا `type` يصل الحدث باسم `message` و `onmessage`.

## شرح المثال

- المسار ما زال يمر بالحارس. `EventSource` في المتصفح يرسل ملفات تعريف الارتباط على نفس الأصل، لا ترويسة `Authorization` حرة. إن كانت رف تعتمد حاملًا فضعوا تصميم الجلسة أو وكيلًا قبل فتح الدفق.
- الجسم لا يفعّل اشتراكًا. الحدث حالة قراءة.
- `finalize` يغلق مؤشرًا أو اشتراك قاعدة عند أي إنهاء.

## ماذا يحدث داخليًا؟

1. Nest يكتب `Content-Type: text/event-stream` ويبقي الاستجابة.
2. كل `next` يصبح كتلة حقول وتنتهي بسطر فارغ إضافي.
3. `event` في المواصفة هو `type` في الكائن.
4. `unsubscribe` عند `close()` من العميل يوقف `interval`.
5. مع `@SseSignal()` يمكن تمرير `signal` إلى `fetch` أعلى حتى يُلغى الطلب إن انتهى الدفق لأي سبب. مستمع `abort` قد يعمل مع دالة التنظيف في Observable؛ اجعلوا الإغلاق مرة واحدة كافيًا.

## مثال عملي

بعد طلب غلاف، الواجهة تفتح الدفق حتى يصل `{ data: { cover: 'ready', projectId } }` ثم تكمل الـ Observable. المعرف في الحمولة يُطابق مشروع المنظمة التي أثبتها الحارس عند فتح الاتصال. لا تثقوا بـ `projectId` داخل حدث لاحق يأتي من عميل. الخادم هو من يكتب الأحداث. إن انقطع المتصفح أثناء فتح جلسة نموذج، `@SseSignal` يرى `aborted` قبل إرجاع Observable وتُغلق الجلسة ثم `return EMPTY`.

## أخطاء شائعة

- إرجاع مصفوفة أو كائنًا عاديًا من المعالج.
- مؤقت بلا `finalize` أو بلا تنظيف الاشتراك.
- اعتبار SSE بديلًا عن ويب هوك Stripe. الدفع يحتاج توقيعًا وإعادة محاولة لا تبويب متصفح.
- فتح الدفق بلا حارس ثم تصفية المنظمة في الحدث فقط.

## متى نستخدمه؟

تحديث أحادي الاتجاه من الخادم إلى تبويب مفتوح: تقدم تصدير، جاهزية غلاف، سجل نشاط.

## متى لا نستخدمه؟

حوار ثنائي الاتجاه؛ ذلك قناة WebSocket. وتكامل مزوّد خارجي؛ ذلك ويب هوك. وطلب ينتهي بجسم واحد؛ مسار REST عادي.

## مقارنة

| الأسلوب | الاتجاه | من يغلق |
| --- | --- | --- |
| REST | طلب واحد | نهاية الاستجابة |
| `@Sse` | خادم → متصفح | العميل أو اكتمال Observable |
| ويب هوك | مزوّد → API | رد 2xx ثم إعادة عند الفشل |

## تمارين

### تمرين سهل

ماذا يعيد معالج `@Sse`؟

<details>
<summary>الحل</summary>

`Observable<MessageEvent>` أو `Promise` يُحَل إلى واحد. غيره لا يطابق عقد الدفق.

</details>

### تمرين متوسط

متى تفحصون `signal.aborted` أثناء الإعداد؟

<details>
<summary>الحل</summary>

بعد عمل غير متزامن وقبل إرجاع Observable. إن انقطع العميل في هذه الفجوة لا يُشترَك في الدفق ولا تعمل دالة تنظيفه. أغلقوا المورد وأعيدوا `EMPTY`.

</details>

### تمرين متقدم

لماذا لا يكفي SSE لتفعيل خطة رف؟

<details>
<summary>الحل</summary>

التبويب قد يُغلق والحدث غير موقّع من Stripe. التفعيل يبقى على الويب هوك ذي التوقيع. الدفق يعرض حالة已知ها الخادم بعد ذلك.

</details>

## أسئلة مراجعة

- من أين `@Sse` و `MessageEvent`؟
- ما حقل `type` على السلك؟
- ماذا يفعل `finalize`؟
- أين `@SseSignal` معرّف؟

## الخلاصة

`@Sse` يبقي استجابة رف مفتوحة ويدفع `MessageEvent`. القطع يلغي الاشتراك. الإشارة تغلق ما خُصص أثناء الإعداد. الدفق عرض حالة لا إثبات دفع.

## ماذا بعد؟

إن صار جزء من رف دالة تُشغَّل عند الطلب في سحابة بلا عملية دائمة، الدرس التالي Serverless.

## المصادر الرسمية

- NestJS — Server-Sent Events: https://docs.nestjs.com/http/server-sent-events
