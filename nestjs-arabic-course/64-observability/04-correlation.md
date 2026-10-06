# ربط السجلات بالطلب

## الهدف من هذا الدرس

وضع معرّف واحد على كل سطر يكتبه طلب رف، حتى تجدوا النشر وPrisma والخطأ في الفلتر معًا.

## المشكلة التي يحلها

خمسة أسطر `published` في نفس الثانية من مكتبات مختلفة. لا تعرفون أيها انتهى بـ 500. الأثر له `traceId` والسجل لا يحمله. الدعم يطلب من الموظف «وقت الخطأ» فقط.

## الفكرة ببساطة

في بداية الطلب خذوا `X-Request-Id` إن كان شكلًا آمنًا أو ولّدوا معرّفًا، واحفظوه في سياق يتبع `async`. كل `Logger` يضيفه. إن وُجد `traceId` من OpenTelemetry أو من Observe فأضيفوه أيضًا. أعيدوا المعرّف في ترويسة الرد وفي جسم 500 إن كان عقدكم يسمح بحقل `requestId` من غير stack.

## التعريف التقني

`AsyncLocalStorage` (أو السياق الذي يبقيه Observe) يعيش مع الطلب. المعترض أو الوسيط يفتح `run` قبل الحارس. لا تثقوا بمعرّف خام فيه أسطر جديدة حتى لا يُحقن السجل. صيغة قصيرة مثل UUID. على Fastify و Express اقرأوا الترويسة من المحول لا تفترضوا `req.headers` فقط إن لففتم الطلب. المهام في الطابور تولّد معرّفًا أو تكمل معرّف الرسالة، لأن لا HTTP هناك. لا تضعوا المعرّف مكان `tenantId`.

## Mental Model

```text
X-Request-Id أو UUID
→ سياق الطلب
→ كل السجلات + ترويسة الرد
traceId بجانبه إن وُجد أثر
```

## أبسط مثال

```ts
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

export const requestStorage = new AsyncLocalStorage<{ requestId: string }>();

export function beginRequest(header: string | undefined) {
  const requestId = header && /^[A-Za-z0-9-]{8,64}$/.test(header) ? header : randomUUID();
  return requestId;
}
```

## شرح المثال

- النمط يرفض المسافات والأسطر.
- الغياب يولّد UUID حتى لا تبقى سجلات بلا مفتاح.
- الدالة لا تفتح السياق؛ من يستدعيها يمرر الناتج إلى `run`.

## ماذا يحدث داخليًا؟

1. الوسيط يقرأ الترويسة ويستدعي `requestStorage.run`.
2. الخدمة تسجّل `{ requestId: requestStorage.getStore()?.requestId }`.
3. فلتر 500 يضيف نفس المعرّف إلى السجل وإلى الجسم `{ code: 'internal_error', requestId }` إن وثّقتم الحقل.
4. الرد يضع `X-Request-Id` حتى يرسله الموظف للدعم.
5. نداء صادر إلى الإشعارات يعيد الترويسة حتى تكمل السلسلة.

## مثال عملي

```ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { requestStorage, beginRequest } from './request-id.js';

@Injectable()
export class RequestIdInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const http = context.switchToHttp();
    const req = http.getRequest<{ headers: Record<string, string> }>();
    const res = http.getResponse<{ setHeader: (k: string, v: string) => void }>();
    const requestId = beginRequest(req.headers['x-request-id']);
    res.setHeader('X-Request-Id', requestId);
    return requestStorage.run({ requestId }, () => next.handle());
  }
}
```

`run` حول `next.handle()` يبقي المعرّف عبر `await` في الخدمة. `enterWith` خارج هذا الغلاف قد يسرّب المعرّف بين الطلبات؛ فضّلوا `run`.

## أخطاء شائعة

- معرّف على singleton في حقل صنف فيتبدل تحت التزامن.
- أخذ المعرّف من الجسم.
- جسم 500 فيه stack بالإضافة إلى المعرّف.
- مهمة الطابور تبحث عن ترويسة HTTP فلا تجد سياقًا وتسجّل `undefined`.

## متى نستخدمه؟

على كل طلب API وعلى كل مهمة تكتب سجلًا. أعيدوه للعميل في الرد.

## متى لا نستخدمه؟

لا تستبدلوا به `traceId` إن كان الأثر يعمل؛ اجمعوهما. ولا تستخدموا معرّف الطلب مفتاح تكرار للدفع.

## مقارنة

| المفتاح | يربط |
| --- | --- |
| `requestId` | أسطر هذا الطلب في السجل |
| `traceId` | spans الأثر |
| `event.id` | تكرار ويب هوك |
| `tenantId` | عزل البيانات لا السجل فقط |

## تمارين

### تمرين سهل

لماذا لا يُقبل أي نص في `X-Request-Id`؟

<details>
<summary>الحل</summary>

حتى لا يُحقن السجل بأسطر أو رموز تحرف الجمع. النمط المحدود أو UUID جديد.

</details>

### تمرين متوسط

لماذا `run` حول `next.handle()` لا حقل على الخدمة؟

<details>
<summary>الحل</summary>

الخدمة singleton وتخدم طلبات متداخلة. الحقل يُكتب من طلب ويُقرأ من آخر. المخزن يتبع السلسلة async لكل طلب.

</details>

### تمرين متقدم

عامل الطابور بلا ترويسة. ماذا تضعون في السجل؟

<details>
<summary>الحل</summary>

معرّف المهمة أو `event.id` من الحمولة، وتفتحون `requestStorage.run` في بداية المعالج. لا تتركوا السطور بلا مفتاح ولا ترثوا سياق آخر طلب HTTP.

</details>

## أسئلة مراجعة

- من أين يأتي المعرّف؟
- أين يُعاد للعميل؟
- لماذا ليس حقلًا على خدمة singleton؟
- ما الفرق بينه وبين `traceId`؟

## الخلاصة

معرّف طلب آمن في سياق async، على كل سطر وفي ترويسة الرد، وبجانب `traceId` إن وُجد أثر. الطابور له معرّفه. المعرّف لا يعزل المستأجرين ولا يكرر الدفع.

## ماذا بعد؟

الإشارات جاهزة لمشروع يجمعها. الفصل التالي يبني رف SaaS على مراحل في دروس، من النطاق حتى Docker والاختبارات، لا كمستودع كود يُسلَّم هنا.

## المصادر الرسمية

- NestJS — Logger: https://docs.nestjs.com/application/logger
- NestJS — Observability: https://docs.nestjs.com/observability/overview
