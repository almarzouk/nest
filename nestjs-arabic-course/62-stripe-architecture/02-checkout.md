# Checkout

## الهدف من هذا الدرس

بدء دفع اشتراك رف بجلسة Stripe Checkout على الخادم، وإعادة رابط للمتصفح، من غير تفعيل الخطة عند فتح رابط النجاح.

## المشكلة التي يحلها

الواجهة تجمع رقم البطاقة وترسله إلى Nest. أو الخادم يعيد `url` ثم يكتب `status: active` لأن الطلب بدأ. الموظف يغلق التبويب قبل الدفع فيصبح الكتالوج على خطة مدفوعة.

## الفكرة ببساطة

`BillingService` ينشئ Customer إن غاب، ثم Checkout Session بنمط `subscription` وسعر الخطة. الرد للمتصفح `{ url }` فقط. `success_url` صفحة شكر تعيد قراءة الحالة من رف. `cancel_url` لا يغيّر الصف. التفعيل حدث لاحق.

## التعريف التقني

SDK على الخادم: `checkout.sessions.create` مع `mode: 'subscription'` و `line_items` فيها `price` من لوحة Stripe لا مبلغ يكتبه المتصفح، و `success_url` و `cancel_url`، و `client_reference_id` أو `metadata.organizationId` من السياق لا من جسم غير موثوق. `customer` المعرّف المخزن. لا تبنوا عناصر البطاقة في رف. المفتاح السري لا يصل إلى الواجهة؛ المفتاح القابل للنشر إن احتجتم منصة أخرى غير Checkout ليس هذا الدرس. Idempotency-Key لإنشاء الجلسة في درس لاحق حتى إعادة ضغط الزر لا تفتح جلسات متنافسة بلا داعٍ.

## Mental Model

```text
زر → POST /billing/checkout → session.url
متصفح → Stripe
success_url → الواجهة تسأل GET /billing
الحالة تتغير فقط بالحدث
```

## أبسط مثال

```ts
export interface CheckoutStart {
  organizationId: string;
  priceId: string;
  successUrl: string;
  cancelUrl: string;
}
```

## شرح المثال

- `priceId` من إعداد الخادم (`STRIPE_PRICE_PRO`) لا من قائمة يختارها العميل بمبلغ حر.
- `organizationId` من سياق المستأجر بعد صلاحية `billing.manage`.
- العناوين من إعدادكم حتى لا يرسل العميل `success_url` إلى نطاقه.

## ماذا يحدث داخليًا؟

1. الحارس يثبت المالك أو `billing.manage`.
2. تُجلب أو تُنشأ `providerCustomerId`.
3. Stripe يعيد `session.url`.
4. تُحفظ `checkoutSessionId` إن أردتم تتبعًا، والحالة تبقى `trial` أو `incomplete`.
5. فتح success لا يستدعي `subscriptions.update` محليًا إلى `active`.

## مثال عملي

```ts
@Post('checkout')
start(@CurrentUser('id') userId: string) {
  return this.billing.startCheckout(currentTenantId(), userId);
}
```

```ts
async startCheckout(organizationId: string) {
  const session = await this.stripe.checkout.sessions.create({
    mode: 'subscription',
    customer: await this.customerFor(organizationId),
    line_items: [{ price: this.priceId, quantity: 1 }],
    success_url: this.successUrl,
    cancel_url: this.cancelUrl,
    metadata: { organizationId },
  });
  if (!session.url) throw new Error('checkout url missing');
  return { url: session.url };
}
```

`this.stripe` عميل الوحدة لا مكتبة الكتب. `metadata` تساعد الحدث على إيجاد المنظمة بالإضافة إلى `customer` المخزن.

## أخطاء شائعة

- كتابة `active` في نهاية `startCheckout`.
- `success_url` من جسم الطلب (توجيه مفتوح).
- سعر يرسله المتصفح.
- انتظار webhook داخل نفس طلب الزر حتى تنتهي المهلة.

## متى نستخدمه؟

عندما تدفع المكتبة لأول خطة أو تغيّر خطة عبر صفحة Stripe المستضافة.

## متى لا نستخدمه؟

لاستعراض الفواتير داخل رف إن وفرتم بوابة العميل (Customer Portal) كأمر منفصل. ولا لتفعيل تجريبي بلا بطاقة؛ ذلك صف `trial` محلي.

## مقارنة

| الخطوة | تغيّر الحالة المحلية؟ |
| --- | --- |
| إنشاء الجلسة | لا إلى `active` |
| فتح success | لا |
| فتح cancel | لا |
| حدث موقّع لاحقًا | نعم |

## تمارين

### تمرين سهل

ماذا يعيد `POST /billing/checkout` إلى الواجهة؟

<details>
<summary>الحل</summary>

`{ url }` لجلسة Stripe. لا يعيد حالة `active`.

</details>

### تمرين متوسط

الموظف وصل إلى success وأغلق الدفع من تبويب آخر قبل الإتمام. ما حالة النشر؟

<details>
<summary>الحل</summary>

الحالة السابقة. success ليست دليل دفع. إن لم يصل حدث نجاح تبقى الخطة كما كانت.

</details>

### تمرين متقدم

لماذا `metadata.organizationId` من السياق لا من `body`؟

<details>
<summary>الحل</summary>

حتى لا تبدأ مكتبة `lib_a` جلسة تُوسَم بـ `lib_b`. الحدث لاحقًا يحدّث من يرتبط بالعميل المخزن، والوسم الكاذب يفتح خطة لجارة.

</details>

## أسئلة مراجعة

- أين تُنشأ الجلسة: متصفح أم خادم؟
- ماذا لا يُكتب عند إنشاء الجلسة؟
- من يحدد `price`؟
- ماذا تفعل صفحة success؟

## الخلاصة

Checkout جلسة على الخادم ورابط للمتصفح. الخطة لا تصبح `active` لأن الرابط رجع. السعر والعميل وعنوان العودة من رف لا من جسم حر.

## ماذا بعد؟

الجلسة بدأت اشتراكًا عند Stripe. الدرس التالي ماذا تنسخون محليًا من كائن الاشتراك.

## المصادر الرسمية

- Stripe — Checkout: https://docs.stripe.com/payments/checkout
- NestJS — Authentication (`CurrentUser`): https://docs.nestjs.com/security/authentication
