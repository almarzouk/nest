# Subscriptions و Payments

## الهدف من هذا الدرس

ربط نشر مشروع رف بخطة محلية، وبدء Checkout من الخادم، ومنع كتابة `active` عندما يفتح المتصفح صفحة النجاح.

## المشكلة التي يحلها

أي مسودة تُنشر فورًا. أو `GET /billing/success` يحدّث الاشتراك لأن الموظف وصل. أو `BooksService` يستورد Stripe فيتعطل النشر مع المزود.

## الفكرة ببساطة

صف `Subscription` للمنظمة يبدأ `trial`. `POST /projects/:id/publish` يسمح به في `trial` و `active` فقط. `POST /billing/checkout` يعيد `{ url }` بمفتاح idempotency. الحالة `active` تُكتب لاحقًا في درس الويب هوك من حدث موقّع. هذه الشريحة توقف المسار الخطأ حتى لو تأخر الحدث.

## التعريف التقني

`PlanPolicy` تقرأ Prisma. `BillingModule` وحدها تعرف SDK. `price` من البيئة لا من الجسم. `metadata.organizationId` من السياق. success URL ثابت في الإعداد. لا رقم بطاقة. إن لم تصل مفاتيح Stripe في التطوير، السياسة تبقى على `trial` الذي يُزرع عند إنشاء المنظمة حتى يعمل النشر في الاختبار بلا شبكة.

## Mental Model

```text
trial عند إنشاء المنظمة
publish يسأل السياسة
checkout يعيد url فقط
success لا يكتب
```

## أبسط مثال

```ts
async publish(organizationId: string, projectId: string) {
  const status = await this.plans.statusOf(organizationId);
  if (status !== 'trial' && status !== 'active') throw new ForbiddenException('plan blocked');
  return this.prisma.project.update({
    where: { id: projectId, organizationId },
    data: { status: 'published' },
  });
}
```

## شرح المثال

- `where` المركب يمنع نشر مشروع الجار.
- `past_due` يسقط في 403.
- لا استدعاء Stripe داخل الدالة.

## ماذا يحدث داخليًا؟

1. إنشاء المنظمة يزرع `trial` في نفس المعاملة إن أمكن.
2. النشر يقرأ الحالة ثم يحدّث المشروع.
3. Checkout ينشئ جلسة ويعيد الرابط ولا يغيّر `trial`.
4. الواجهة على success تستدعي `GET /billing` وترى الحالة القديمة حتى يصل الحدث.
5. درس الويب هوك لاحقًا هو الكاتب لـ `active`.

## مثال عملي

```ts
@Get('success')
success() {
  return this.billing.view(currentTenantId());
}
```

`view` قراءة صف. لا `update`. إن غاب المسار أصلًا فالواجهة تتحدث إلى `GET /billing` فقط. المهم ألا يوجد `data: { status: 'active' }` هنا.

## أخطاء شائعة

- تفعيل في `success`.
- زر النشر ينتظر Stripe.
- تجربة محلية بلا صف `trial` فكل نشر 403 بلا تفسير.
- مفتاح idempotency من `Date.now()`.

## متى نستخدمه؟

بعد وجود المشاريع. انشروا من `trial` في الاختبارات حتى لا تحتاج كل بيئة مفتاحًا.

## متى لا نستخدمه؟

لا تبنوا بوابة فواتير كاملة. ولا تمنعوا `GET /projects` عند `past_due`.

## مقارنة

| الفعل | يغيّر `Subscription.status` إلى active؟ |
| --- | --- |
| إنشاء منظمة | يزرع `trial` |
| Checkout | لا |
| success | لا |
| publish | يقرأ فقط |

## تمارين

### تمرين سهل

ماذا يعيد Checkout للواجهة؟

<details>
<summary>الحل</summary>

`{ url }` والصف المحلي يبقى `trial` أو `incomplete` لا `active`.

</details>

### تمرين متوسط

`past_due` ثم `POST publish`. ما الرمز وهل يتغير المشروع؟

<details>
<summary>الحل</summary>

403 ولا `update`. القراءة ما زالت 200.

</details>

### تمرين متقدم

اختبار بلا شبكة Stripe. كيف يُنشر مشروع؟

<details>
<summary>الحل</summary>

زرع `Subscription.status = trial` للمنظمة ثم استدعاء النشر. لا حاجة إلى Checkout في هذا الاختبار. اختبار Checkout منفصل يتوقع `{ url }` فقط عندما يكون المفتاح موجودًا.

</details>

## أسئلة مراجعة

- من يزرع `trial`؟
- أي حالات تسمح بالنشر؟
- ماذا لا يفعل success؟
- أين يُمنع استيراد Stripe؟

## الخلاصة

النشر مشروط بحالة محلية. Checkout رابط. صفحة النجاح قراءة. `active` ليس نتيجة زر ولا نتيجة عودة المتصفح.

## ماذا بعد؟

دعوة موظف لا تنتظر البريد. الدرس التالي الإشعارات في المشروع.

## المصادر الرسمية

- Stripe — Checkout: https://docs.stripe.com/payments/checkout
- NestJS — Modules: https://docs.nestjs.com/modules
