# Docker و Tests

## الهدف من هذا الدرس

تشغيل رف في حاوية مع Postgres و Redis، وإثبات اختبارين: قائمة فارغة 200، ومنظمة لا تقرأ مشروع الأخرى. ESM يستخدم Vitest.

## المشكلة التي يحلها

الاختبار الوحيد `should be defined` على الخدمة. أو صورة Docker تشغّل `nest start` وتبني داخل الإنتاج. أو Jest على مشروع `"type": "module"` فيفشل الاستيراد. أو الاختبار يتصل بقاعدة التطوير المشتركة فيمسح مكتبات حقيقية.

## الفكرة ببساطة

`Dockerfile` متعدد المراحل: بناء ثم `node dist/main.js`. Compose يرفع Postgres و Redis و API. الصحة `ready` تنتظر القاعدة. الاختبارات على قاعدة اختبار. مشروع Nest 12 الافتراضي ESM مع Vitest؛ CommonJS هو الذي يبقى على Jest. لا مستودع يُنشأ في مجلد الدورة؛ المقتطف يُنسخ عندما تبنون رف عندكم.

## التعريف التقني

اختبار وحدة لـ `PlanPolicy` و `projects.create` بمنفذ مزدوج إن شئتم. اختبار تطبيق واحد على الأقل بـ `supertest` أو عميل HTTP: سجّلوا مستخدمين، منظمتين، مشروعًا في الأولى، واطلبوه بترويسة الثانية فتوقعوا 404، واطلبوا قائمة الثانية فتوقعوا 200 و `data: []`. لا تختبروا Stripe الحي؛ ازرعوا `trial`. Vitest يشغّل ملفات `*.spec.ts`. لا تضعوا `STRIPE_SECRET_KEY` الحقيقي في Compose العام.

## Mental Model

```text
compose: postgres + redis + api
اختبار: org A ترى مشروعها
        org B قائمة فارغة 200 و 404 على معرّف A
```

## أبسط مثال

```dockerfile
FROM node:22 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./
RUN npm ci --omit=dev
CMD ["node", "dist/main.js"]
```

## شرح المثال

- مرحلة التشغيل بلا سلسلة البناء الكاملة إن اكتفيتم بـ `dist`. إن احتاج Prisma المحرك، انسخوا المخطط وولّدوا العميل في مرحلة مناسبة حتى لا تفشل الإقلاع.
- `node:22` يوافق خط Nest 12 (22.12 أو أحدث على هذا الخط).
- الأمر `node` لا `nest start` حتى لا يعيد البناء في الإنتاج.

## ماذا يحدث داخليًا؟

1. Compose يشغّل Postgres ثم API بعد الجاهزية.
2. الترحيل أمر منفصل لا يُخفى داخل طلب القارئ: `prisma migrate deploy` في مهمة إطلاق.
3. `GET /health/ready` ينجح عندما تقبل القاعدة الاتصال.
4. اختبار العزل يزرع صفين منظمة.
5. الفشل يعني `where` ناقصًا لا «المختبِر».

## مثال عملي

```ts
import { describe, expect, it } from 'vitest';

describe('project isolation', () => {
  it('hides other organizations', async () => {
    const created = await asOrg('org_a').post('/projects').send({ title: 'موسم' });
    expect(created.status).toBe(201);
    const list = await asOrg('org_b').get('/projects');
    expect(list.status).toBe(200);
    expect(list.body.data).toEqual([]);
    const foreign = await asOrg('org_b').get(`/projects/${created.body.id}`);
    expect(foreign.status).toBe(404);
  });
});
```

`asOrg` يضع الإثبات و `X-Tenant-Id`. هذا اختبار تطبيق لا وحدة منعزلة. جهّزوا `app` كما في توثيق اختبار Nest.

## أخطاء شائعة

- Jest على ESM افتراضي بلا إعداد، بينما Vitest هو الافتراضي.
- اختبار يتوقع 404 للقائمة الفارغة.
- Compose يشغّل نجاح Stripe وهمي يكتب `active`.
- صورة واحدة فيها أسرار إنتاج.

## متى نستخدمه؟

كآخر شريحة عندما تعمل المسارات محليًا. العزل أول اختبار تطبيق لا آخره.

## متى لا نستخدمه؟

لا تنتظروا Docker لتجريب `PlanPolicy`. الوحدة أسرع. ولا تضيفوا عنقود Kubernetes في هذا الفصل.

## مقارنة

| الفحص | الأداة |
| --- | --- |
| سياسة الخطة | وحدة Vitest |
| عزل المنظمات | تطبيق على قاعدة اختبار |
| إقلاع حقيقي | Compose وصحة ready |
| ESM | Vitest لا Jest |

## تمارين

### تمرين سهل

ماذا تتوقع قائمة منظمة بلا مشاريع؟

<details>
<summary>الحل</summary>

200 و `data: []` لا 404.

</details>

### تمرين متوسط

لماذا `CMD` هو `node dist/main.js`؟

<details>
<summary>الحل</summary>

لأن البناء تم في المرحلة الأولى. `nest start` يعيد البناء ويحتاج سلسلة التطوير.

</details>

### تمرين متقدم

اكتبوا حالة اختبار ثالثة تمنع success من تفعيل الخطة.

<details>
<summary>الحل</summary>

ازرعوا `trial`، استدعوا مسار success، توقعوا بقاء `trial` أو `incomplete` وأن النشر ما زال مسموحًا بسبب التجربة لا لأن المسار كتب `active`. لا اتصال بـ Stripe.

</details>

## أسئلة مراجعة

- أي مشغّل مع ESM؟
- ماذا يثبت اختبار المنظمتين؟
- ماذا لا يُكتب في success داخل الاختبار؟
- أين يُشغَّل الترحيل؟

## الخلاصة

الحاوية تشغّل `dist` بجانب Postgres و Redis. Vitest يثبت أن القائمة الفارغة 200 وأن منظمة لا تقرأ مشروع الأخرى. صفحة النجاح لا تُختبر كزر تفعيل.

## ماذا بعد؟

نطاق رف اكتمل على الورق حتى الاختبار. الفصل التالي أنماط الاعتمادية عندما لا تكفي إعادة المحاولة البسيطة: مفاتيح، صندوق صادر كامل، وتدفقات وقوف.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
- NestJS — Deployment: https://docs.nestjs.com/deployment
- Vitest: https://vitest.dev/
