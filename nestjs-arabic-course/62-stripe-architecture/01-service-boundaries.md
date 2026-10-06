# حدود الخدمة

## الهدف من هذا الدرس

عزل Stripe عن كتالوج رف: وحدة فوترة تتحدث إلى المزود، ووحدة كتب تقرأ حالة محلية فقط.

## المشكلة التي يحلها

`BooksService.publish` يستدعي `stripe.subscriptions.retrieve` في كل طلب. تعطل Stripe يوقف القراءة والكتابة. مفاتيح API تظهر في وحدة الواجهة لأن «الدفع صفحة». رقم البطاقة يُطلب في DTO عندكم.

## الفكرة ببساطة

رف تملك المنظمات والكتب والحالة `trial | active | past_due`. Stripe تملك العميل والبطاقة والفواتير. الحدود: أوامر مثل «ابدأ Checkout» و«فسّر حدثًا موقّعًا». لا استعلام شبكة داخل مسار الكتاب. السر `STRIPE_SECRET_KEY` وسر الويب هوك على الخادم فقط.

## التعريف التقني

وحدة `BillingModule` تصدّر `BillingService` تغلّف SDK. `BooksModule` تستورد سياسة تقرأ `Subscription` من Prisma لا عميل Stripe. `success_url` صفحة في الواجهة. معالجها في API لا يكتب `active`. الحقول المحلية: `providerCustomerId`, `providerSubscriptionId`, `plan`, `status`. لا PAN. idempotency لاحقًا على إنشاء الجلسة حتى لا تفتح جلستين عند إعادة الطلب.

## Mental Model

```text
كتب  →  SELECT subscription.status
فوترة →  Stripe API + webhook
لا سهم من الكتب إلى SDK
```

## أبسط مثال

```ts
@Injectable()
export class PlanPolicy {
  constructor(private readonly prisma: PrismaService) {}

  async canPublish(organizationId: string) {
    const sub = await this.prisma.subscription.findUnique({ where: { organizationId } });
    return sub?.status === 'active' || sub?.status === 'trial';
  }
}
```

## شرح المثال

- لا import لـ `stripe` في هذا الملف.
- الغياب أو الحالة الأخرى تمنع النشر على الحد.
- الاختبار يزرع صفًا محليًا ولا يحاكي شبكة.

## ماذا يحدث داخليًا؟

1. أمر الدفع يذهب إلى `BillingService`.
2. Stripe يعيد معرّفات تُحفظ محليًا.
3. الحدث الموقّع لاحقًا يحدّث `status`.
4. `publish` يقرأ الصف.
5. انقطاع Stripe بعد ذلك لا يغيّر قرار النشر إلا عندما تتغير الحالة المخزنة.

## مثال عملي

```ts
export interface BillingPort {
  startCheckout(organizationId: string): Promise<{ url: string }>;
}

export interface ClockPolicy {
  allowsPublish(organizationId: string): Promise<boolean>;
}
```

الواجهتان تمنعان استيراد SDK في الكتب. وحدة الفوترة تنفّذ الأولى. سياسة بسيطة تنفّذ الثانية. اختبار الكتب يستبدل الثانية بذاكرة.

## أخطاء شائعة

- مفتاح سري في حزمة الواجهة.
- تفعيل الخطة من مسار success.
- خلط سعر الكتاب `priceCents` مع `price_...` في Stripe.
- جعل كل مسار `await stripe` «ليبقى متزامنًا مع الحقيقة». الحقيقة التشغيلية نسختكم بعد الحدث.

## متى نستخدمه؟

من أول مفتاح Stripe. الوحدة حد مراجعة: لا SDK خارج `billing/`.

## متى لا نستخدمه؟

لا تعزلوا حتى يختفي معرّف العميل؛ الكتب تحتاج الحالة لا الفاتورة. ولا تبنوا محفظة بطاقات داخل رف.

## مقارنة

| | داخل رف | داخل Stripe |
| --- | --- | --- |
| الكتب | نعم | لا |
| رقم البطاقة | لا | نعم |
| حالة الخطة للعمل | نسخة محلية | المصدر الذي يصل عبر حدث |
| مفتاح API | خادم الفوترة | — |

## تمارين

### تمرين سهل

هل `GET /books` يتصل بـ Stripe؟

<details>
<summary>الحل</summary>

لا. يقرأ القاعدة. Stripe يُستدعى من وحدة الفوترة عند بدء الدفع وعند الحدث.

</details>

### تمرين متوسط

لماذا يوقف تعطل Stripe النشر إن كان داخل `BooksService`؟

<details>
<summary>الحل</summary>

لأن كل نشر ينتظر الشبكة. الحالة المحلية تجعل النشر يعمل ما دامت النسخة كافية، وتتأخر عن المزود فقط حتى يصل الحدث.

</details>

### تمرين متقدم

ارسموا ما يُمنع استيراده في `books.module.ts`.

<details>
<summary>الحل</summary>

حزمة `stripe` ومفتاح السر. المسموح: `PlanPolicy` أو منفذ يقرأ الحالة. إنشاء الجلسة يبقى في `BillingModule`.

</details>

## أسئلة مراجعة

- من يملك البطاقة؟
- ماذا تستورد وحدة الكتب؟
- أين يعيش المفتاح السري؟
- ما النسخة التي يقرأها النشر؟

## الخلاصة

Stripe خلف وحدة فوترة. الكتالوج يقرأ حالة محلية. المفاتيح على الخادم، وصفحة النجاح ليست أمر تفعيل.

## ماذا بعد؟

أول أمر عبر الحد: إنشاء جلسة Checkout. الدرس التالي ماذا تعيدون للمتصفح وماذا لا تكتبونه بعد.

## المصادر الرسمية

- Stripe — Checkout: https://docs.stripe.com/payments/checkout
- NestJS — Configuration: https://docs.nestjs.com/application/configuration
