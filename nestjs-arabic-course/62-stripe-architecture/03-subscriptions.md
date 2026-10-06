# Subscriptions

## الهدف من هذا الدرس

تمثيل اشتراك مكتبة رف كصف محلي يطابق كائن Stripe، وتحديثه من أحداث لا من صفحة العودة.

## المشكلة التي يحلها

الفريق يخزّن «مدفوع حتى» يحسبه المتصفح. أو ينسخ كل فاتورة إلى جدول الكتب. عند `past_due` يستمر النشر لأن الحقل المحلي لم يُحدَّث منذ Checkout. إلغاء المكتبة في بوابة Stripe لا يصل إلى رف.

## الفكرة ببساطة

صف واحد لكل منظمة: معرّف العميل، معرّف الاشتراك، `price`/`plan`, `status` من مجموعة تفهمونها. الأحداث `customer.subscription.updated` و `invoice.paid` و `invoice.payment_failed` هي التي تكتب الصف. النشر يقرأ `status` فقط.

## التعريف التقني

اربطوا `customer` بـ `organizationId` عند أول Checkout. عند الحدث، جدوا المنظمة بهذا المعرّف لا بحقل JSON يختاره المرسل من غير ربط. الحالات التي تكفي رف: `trialing` إن بقي عند المزود، `active`, `past_due`, `canceled`. لا تحفظوا وسيلة الدفع. التجديد الناجح يبقي `active`. فشل التحصيل يضع `past_due` ويمنع النشر حسب السياسة. الإلغاء `canceled` ويبقي الصف للمراجعة لا `DELETE`.

## Mental Model

```text
Stripe subscription.id  ↔  Subscription.providerSubscriptionId
status على الكائن        ↔  Subscription.status
كتب                      →  تقرأ status فقط
```

## أبسط مثال

```ts
export function mapStripeStatus(status: string) {
  if (status === 'active' || status === 'trialing') return status === 'trialing' ? 'trial' : 'active';
  if (status === 'past_due' || status === 'unpaid') return 'past_due';
  if (status === 'canceled' || status === 'incomplete_expired') return 'canceled';
  return 'incomplete';
}
```

## شرح المثال

- الخريطة صريحة حتى لا يمر `unpaid` كسلسلة غريبة تسمح بالنشر.
- `trialing` عند Stripe يقابل `trial` محليًا إن هذا اسم عقدكم.
- أي حالة مجهولة `incomplete` تمنع النشر إلى أن تفهموها. الفشل المغلق أسلم من السماح.

## ماذا يحدث داخليًا؟

1. Checkout ينشئ اشتراكًا عند Stripe بحالة قد تكون `incomplete` حتى تُدفع الفاتورة.
2. الحدث يحمل `customer` و `status`.
3. المعالج يجد المنظمة ويكتب `mapStripeStatus`.
4. `PlanPolicy` في الطلب التالي ترى القيمة.
5. لا مسار success يكتب هذه الدالة.

## مثال عملي

```ts
async syncFromStripe(customerId: string, stripeStatus: string, subscriptionId: string) {
  const row = await this.prisma.subscription.findFirst({ where: { providerCustomerId: customerId } });
  if (!row) throw new Error('unknown customer');
  return this.prisma.subscription.update({
    where: { organizationId: row.organizationId },
    data: { status: mapStripeStatus(stripeStatus), providerSubscriptionId: subscriptionId },
  });
}
```

الاستدعاء من معالج الويب هوك بعد التوقيع والقيد، لا من المتحكم الذي يعيد `{ url }`.

## أخطاء شائعة

- `DELETE` الصف عند الإلغاء فيضيع من كانت مكتبته.
- السماح بالنشر لكل حالة غير `canceled`.
- تحديث المنظمة التي في `metadata` حتى لو خالفت `customer` المخزن.
- حساب `past_due` في الواجهة من تاريخ محلي بلا حدث.

## متى نستخدمه؟

طوال عمر المكتبة المدفوعة. الصف يعيش مع المنظمة.

## متى لا نستخدمه؟

لا تعكسوا كل `invoice.line` في جدول الكتب. الفاتورة تبقى عند Stripe وتُعرض برابط البوابة إن لزم.

## مقارنة

| حدث | حالة محلية تقريبية |
| --- | --- |
| اشتراك `active` | `active` |
| `past_due` / `unpaid` | `past_due` |
| `canceled` | `canceled` |
| فتح success URL | بلا كتابة |

## تمارين

### تمرين سهل

من أين تُكتب `providerSubscriptionId` أول مرة بشكل موثوق؟

<details>
<summary>الحل</summary>

من حدث Stripe بعد التوقيع، أو من استجابة إنشاء تعرفون أنها من SDK الخادم، لا من استعلام success في المتصفح.

</details>

### تمرين متوسط

لماذا `unpaid` لا يبقى نصًا خامًا في السياسة؟

<details>
<summary>الحل</summary>

حتى لا تنسوه في شرط `=== 'active'`. الخريطة تضعهم في `past_due` والسياسة ترفض النشر بقائمة قصيرة.

</details>

### تمرين متقدم

`metadata.organizationId` يقول `lib_b` و `customer` المخزن لـ `lib_a`. أي صف يُحدَّث؟

<details>
<summary>الحل</summary>

صف `lib_a` المرتبط بالعميل، أو ترفضون الحدث وتسجلونه إن كان التعارض غير متوقع. لا تحدّثون `lib_b` من metadata وحدها.

</details>

## أسئلة مراجعة

- ماذا يُنسخ من كائن الاشتراك؟
- أي حالات تمنع النشر في هذا التصميم؟
- لماذا لا يُحذف الصف عند الإلغاء؟
- أي مسار لا يستدعي `syncFromStripe`؟

## الخلاصة

الاشتراك صف منظمة يطابق حالة Stripe عبر خريطة صريحة. التحديث من معرّف العميل المخزن بعد حدث موقّع. النشر يقرأ الخريطة المحلية لا لوحة Stripe.

## ماذا بعد؟

الخريطة لا تعمل بلا مسار يستقبل الحدث. الدرس التالي ويب هوك Stripe على الجسم الخام.

## المصادر الرسمية

- Stripe — Subscriptions: https://docs.stripe.com/billing/subscriptions/overview
- NestJS — Webhooks: https://docs.nestjs.com/http/webhooks
