# Webhooks

## الهدف من هذا الدرس

توصيل وارد Stripe وصادر `project.published` في رف: جسم خام، توقيع، رد بعد الحفظ، ومعرّف ثابت.

## المشكلة التي يحلها

صفحة النجاح ما زالت مغرية لكتابة `active`. أو الصادر يُرسل داخل `publish` فينتظر الشريك. أو كل إعادة ترسل معرّفًا جديدًا.

## الفكرة ببساطة

`POST /webhooks/stripe` عام يتحقق بـ `constructEvent(rawBody, signature, secret)` ثم يدرج `event.id` ويحدّث الاشتراك. `publish` بعد الالتزام يدرج تسليمًا صادرًا بنفس أسلوب الصندوق المبسط: صف `pending` لا `fetch` متزامن. التوقيع الصادر على البايتات. 200 للوارد بعد الحفظ. المكرر 200 بلا أثر ثانٍ.

## التعريف التقني

`rawBody: true` في `bootstrap`. سر الويب هوك غير مفتاح API. المنظمة من `providerCustomerId` لا من جسم حر. الصادر: `WebhookDelivery` بمعرّف حدث ثابت و `organizationId`. لا تفعيل من `GET /billing/success`. التفاصيل المعمارية سُبقت في فصلي الويب هوك و Stripe؛ هنا مكانها في مسار رف فقط.

## Mental Model

```text
Stripe → rawBody → حدث مرة واحدة → status
publish → صف delivery → عامل يوقّع ويرسل
success → قراءة فقط
```

## أبسط مثال

```ts
const event = this.stripe.webhooks.constructEvent(req.rawBody!, signature, secret);
```

بعد فحص `rawBody` ووجود الترويسة. الفشل 400.

## شرح المثال

- لا `JSON.stringify(req.body)`.
- النوع يُعالج في الخدمة لا في المتحكم الطويل.
- الرد بعد الإدراج.

## ماذا يحدث داخليًا؟

1. حدث `customer.subscription.updated` يمر `constructEvent`.
2. `markEvent` إما يطبّق `mapStripeStatus` أو يعود كمكرر.
3. `publish` يلتزم المشروع والتدقيق ثم صف التسليم `project.published`.
4. العامل يرسل HMAC و `X-Raf-Event-Id`.
5. 503 من الشريك يؤجل نفس المعرّف. 400 توقيع يتوقف.

## مثال عملي

```ts
await tx.webhookDelivery.create({
  data: { id: eventId, organizationId, type: 'project.published', status: 'pending' },
});
```

داخل معاملة النشر إن أردتم ألا يضيع التبليغ، وهذا صندوق مبسّط. العامل خارج المعاملة. لا تضعوا هذا الإدراج في مسار success.

## أخطاء شائعة

- كتابة `active` في success لأن الويب هوك «سيأتي لاحقًا» ثم نسيان المسار.
- صادر بلا منظمة في الجدول.
- رد 200 قبل INSERT.
- معرّف جديد لكل محاولة.

## متى نستخدمه؟

عندما تعمل مفاتيح Stripe وعندما تسجّل مكتبة عنوان شريك. الاختبار يحاكي Buffer موقّعًا بلا شبكة إن أمكن.

## متى لا نستخدمه؟

لا تستقبلوا Stripe على مسار الجلسة. ولا ترسلوا الصادر قبل وجود مشروع `published`.

## مقارنة

| المسار | يكتب active | يرسل للشريك |
| --- | --- | --- |
| success | لا | لا |
| ويب هوك Stripe | حسب الحالة الموقّعة | لا |
| publish | لا | يدرج تسليمًا |

## تمارين

### تمرين سهل

ماذا يُمرَّر إلى `constructEvent`؟

<details>
<summary>الحل</summary>

`rawBody` و `Stripe-Signature` وسر الوجهة.

</details>

### تمرين متوسط

إعادة نفس `evt_1` بعد تحديث ناجح. كم مرة تتغير الخطة؟

<details>
<summary>الحل</summary>

مرة. الثانية قيد فريد و 200 بلا `update`.

</details>

### تمرين متقدم

لماذا صف التسليم داخل معاملة النشر أقرب من `fetch` بعدها؟

<details>
<summary>الحل</summary>

الصف يُخلق مع الحقيقة. `fetch` قد لا يحدث إن قُتلت العملية، وإن حدث قبل الالتزام فقد يبلّغ عن نشر تراجع. العامل يسلّم الصف لاحقًا.

</details>

## أسئلة مراجعة

- أي مسار يكتب حالة الخطة؟
- ماذا يُدرج عند النشر للشريك؟
- لماذا المعرّف ثابت؟
- أين `rawBody`؟

## الخلاصة

وارد Stripe على الجسم الخام ومعرّف مرة واحدة. صادر النشر صف داخل المعاملة يسلّمه عامل. صفحة النجاح لا تكتب ولا ترسل.

## ماذا بعد؟

شخص منصة يرى المكتبات للمشاكل لا عبر ترويسة عميل. الدرس التالي الإدارة.

## المصادر الرسمية

- NestJS — Raw body: https://docs.nestjs.com/faq/raw-body
- Stripe — Webhooks: https://docs.stripe.com/webhooks
