# Webhook

## الهدف من هذا الدرس

استقبال أحداث Stripe في رف على مسار عام، والتحقق بسر الويب هوك على `rawBody`، ثم مزامنة الاشتراك.

## المشكلة التي يحلها

المسار يستخدم `@Body()` ثم `constructEvent` على نص أُعيد تسلسله فيفشل التوقيع الصحيح. أو ينجح أي JSON فيه `type: customer.subscription.updated` لأن السر غير مستخدم. أو يكتب `active` من مسار success «إلى أن نربط الويب هوك».

## الفكرة ببساطة

Stripe يوقّع البايتات ويضع `Stripe-Signature`. `stripe.webhooks.constructEvent(rawBody, signature, webhookSecret)` يرمي إن فسد التوقيع. بعدها تفرّعون على `event.type` وتستدعون مزامنة الاشتراك. تردون 200 بعد الحفظ. هذا نفس شكل الوارد العام، بأداة Stripe لا بـ HMAC مكتوب يدويًا إن استخدمتم SDK.

## التعريف التقني

`rawBody: true` في `NestFactory.create` كما في صفحة Raw body. لا `bodyParser: false`. السر `STRIPE_WEBHOOK_SECRET` غير مفتاح API السري؛ لكل وجهة ويب هوك سر. `@Public()` على المسار مع هذا التحقق. الأحداث التي تهم رف في هذا الفصل: `checkout.session.completed` لربط المعرّفات إن نقصت، و `customer.subscription.updated` و `customer.subscription.deleted` و `invoice.payment_failed`. `constructEvent` يرفض الطابع القديم حسب تسامح المكتبة. لا تحدّثوا الحالة من غير أن يمر البناء.

## Mental Model

```text
Stripe-Signature + Buffer
→ constructEvent
→ switch type
→ syncFromStripe
→ 200
```

## أبسط مثال

```ts
@Post()
@HttpCode(200)
handle(@Req() req: RawBodyRequest<Request>, @Headers('stripe-signature') signature: string) {
  const event = this.stripe.webhooks.constructEvent(req.rawBody!, signature, this.secret);
  return { type: event.type };
}
```

أضيفوا الاستيرادات من `@nestjs/common` ونوع `Request` من `express`. `rawBody!` بعد فحص الغياب برمي 400.

## شرح المثال

- `constructEvent` هو التحقق. لا تثقوا بـ `event.type` قبله.
- الرد 200 هنا مبكر للشرح. الإنتاج يحفظ ثم يرد، كما في درس إعادة المحاولة.
- المفتاح في الترويسة اسمه `stripe-signature`.

## ماذا يحدث داخليًا؟

1. Stripe يرسل الحدث إلى عنوانكم.
2. Nest يبقي Buffer.
3. `constructEvent` يتحقق من HMAC والطابع.
4. المعالج يجد المنظمة عبر `customer` ويستدعي `mapStripeStatus`.
5. الفشل قبل الحفظ يعيد 500 حتى يعيد Stripe. التوقيع الفاسد 400.

## مثال عملي

```ts
async onEvent(event: { type: string; data: { object: { customer?: string; status?: string; id: string } } }) {
  if (event.type === 'customer.subscription.updated' || event.type === 'customer.subscription.deleted') {
    const object = event.data.object;
    if (!object.customer || !object.status) return;
    await this.subs.syncFromStripe(object.customer, object.status, object.id);
  }
}
```

`checkout.session.completed` يمكنه حفظ `subscription` و `customer` إن لم يُحفظا، من غير وضع `active` إن كانت حالة الاشتراك ما زالت غير مدفوعة. لا تشتقوا `active` من اسم الحدث `completed` وحده.

## أخطاء شائعة

- سر API مكان سر الويب هوك.
- تفعيل الخطة داخل فرع success في الواجهة لأن الحدث تأخر.
- `constructEvent(JSON.stringify(req.body), ...)`.
- معالجة كل الأنواع بتحديث أعمى إلى `active`.

## متى نستخدمه؟

كقناة وحيدة لكتابة `Subscription.status` بعد إطلاق Checkout.

## متى لا نستخدمه؟

لا تستقبلوا الحدث على مسار جلسة الموظف. ولا تستخدموا وجهة اختبار الإنتاج بسر التطوير.

## مقارنة

| المسار | يكتب `active`؟ |
| --- | --- |
| `POST /billing/checkout` | لا |
| `GET /billing/success` | لا |
| ويب هوك بعد `constructEvent` | حسب `mapStripeStatus` فقط |

## تمارين

### تمرين سهل

ما الترويسة التي يقرأها `constructEvent` مع الـ Buffer؟

<details>
<summary>الحل</summary>

`Stripe-Signature`، مع سر الوجهة `STRIPE_WEBHOOK_SECRET`، لا مفتاح السر العام للحساب وحده.

</details>

### تمرين متوسط

لماذا `checkout.session.completed` لا يكفي ليضع `active` دائمًا؟

<details>
<summary>الحل</summary>

اكتمال الجلسة قد يسبق حالة الاشتراك التي تفهمونها، أو يرتبط بدفع لم يُحصَّل. اقرأوا حالة الاشتراك من الحدث المناسب ومرّروها على `mapStripeStatus`.

</details>

### تمرين متقدم

التوقيع صحيح والعميل غير موجود في رف. ماذا تردون ولماذا؟

<details>
<summary>الحل</summary>

لا تحدّثوا منظمة عشوائية. سجّلوا الحدث المجهول. رد 200 بعد تسجيل آمن يمنع إعادات لا تنتهي إن كان الحدث ليس لكم، أو 500 إن كان الخلل مؤقتًا في القاعدة. لا تفتحوا خطة من metadata فقط.

</details>

## أسئلة مراجعة

- ماذا يُمرَّر إلى `constructEvent` غير السر؟
- لماذا سر الويب هوك غير مفتاح API؟
- أي مسار لا يكتب الحالة؟
- متى 400 ومتى 500؟

## الخلاصة

ويب هوك Stripe يتحقق بـ `constructEvent` على `rawBody` و `Stripe-Signature`. بعدها تُزامَن الحالة من نوع الحدث. success URL لا تستدعي هذا المسار نيابة عنكم.

## ماذا بعد؟

إعادة ضغط زر Checkout أو إعادة Stripe للحدث تحتاج مفاتيح. الدرس التالي idempotency على أوامر Stripe وعلى معرّف الحدث.

## المصادر الرسمية

- Stripe — Webhooks: https://docs.stripe.com/webhooks
- NestJS — Raw body: https://docs.nestjs.com/faq/raw-body
