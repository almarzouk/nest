# أمثلة عملية

دار النيل تضغط «ترقية» ولا تصبح `active` إلا بحدث.

## المثال 1 — الحد

`BooksService` تستدعي `PlanPolicy` التي تقرأ `trial`. لا استيراد لـ `stripe` في وحدة الكتب. `STRIPE_SECRET_KEY` في بيئة وحدة الفوترة.

## المثال 2 — الجلسة

`POST /billing/checkout` للمالك ينشئ جلسة `mode: 'subscription'` بمفتاح `checkout:org_nile:pro` ويعيد `{ url }`. فتح `success_url` يستدعي `GET /billing` فيرى `trial` أو `incomplete`. لا كتابة.

## المثال 3 — الحدث

`constructEvent(req.rawBody, header, STRIPE_WEBHOOK_SECRET)` ثم `customer.subscription.updated` بحالة `active`. `markEvent` يدرج `evt_1` و `syncFromStripe` يكتب `active` لمنظمة عميل Stripe المخزن. إعادة `evt_1` ترد 200 بلا بريد ثانٍ. `past_due` لاحقًا يوقف النشر ويترك القراءة.
