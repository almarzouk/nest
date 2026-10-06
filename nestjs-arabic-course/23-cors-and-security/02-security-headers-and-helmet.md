# Security Headers و Helmet

## الهدف من هذا الدرس

تفعيل رؤوس الأمان التي يوفّرها Nest منذ `12.1` عبر `useSecurityHeaders`، وربطها بسلوك Helmet.

## المشكلة التي يحلها

صفحة HTML أو Swagger تُحمَّل في إطار موقع آخر، أو يخمّن المتصفح نوع ملف فيُنفّذ محتوى، أو تسريب `Referer` فيه مسار حساس. الاعتماد على مقال `helmet()` القديم بأسماء خيارات مثل `hsts` ينكسر مع الواجهة الحالية.

## الفكرة ببساطة

`app.useSecurityHeaders()` يضع مجموعة رؤوس افتراضية تماثل Helmet 8، على Express و Fastify، بلا حزمة إضافية. منها `Content-Security-Policy` و `X-Frame-Options` و `Strict-Transport-Security`، ويحذف `X-Powered-By`. من يملك Helmet أصلًا يقدر أن يبقيه. الأسماء الحالية في Nest هي الأسماء الجديدة لا الأسماء القديمة مثل `frameguard`.

## التعريف التقني

```ts
const app = await NestFactory.create(AppModule);
app.useSecurityHeaders();
await app.listen(3000);
```

استدعِها بعد `create` وقبل `listen` أو `init`. نداء متأخر أو مكرر يرمي. الرؤوس على استجابات المسارات و `404` والأخطاء ورفض CSRF.

تعطيل رأس: مرّر `false`. تخصيص: كائن، مثل `xFrameOptions: { action: "deny" }` أو `strictTransportSecurity: { maxAge: 63072000, preload: true }`. اسم قديم مثل `hsts` يُرفض مع تلميح إلى `strictTransportSecurity`.

`Content-Security-Policy` الافتراضي يقصر السكربت على `'self'` مع تفاصيل الصفحة. `'unsafe-inline'` في `script-src` يضعف الحماية من XSS. Swagger الذي تقدّمه `@nestjs/swagger` يعمل مع الافتراضي لأن أصوله من تطبيقك. GraphiQL في التطوير قد يحتاج استثناء `unpkg` كما في الصفحة، لا في الإنتاج.

`X-XSS-Protection: 0` في الجدول الافتراضي يطفئ مرشّحًا قديمًا كان مصدر ثغرات، وليس تفعيل حماية XSS حديثة. الحماية الحديثة CSP وتهريب المخرجات.

Helmet المباشر ما زال موثّقًا: `app.use(helmet())` على Express و `@fastify/helmet` على Fastify. إن انتقلت إلى `useSecurityHeaders` فالأسماء متوافقة مع Helmet الحديث باستثناء دوال CSP لكل طلب، فهي غير مدعومة في الواجهة المدمجة.

## Mental Model

```text
useSecurityHeaders()
  CSP تحد السكربت
  X-Frame-Options تحد الإطار
  HSTS تطلب HTTPS لاحقًا
  X-Powered-By يُحذف
```

## أبسط مثال

```ts
app.useSecurityHeaders({
  referrerPolicy: { policy: "strict-origin-when-cross-origin" },
  xFrameOptions: { action: "sameorigin" },
});
```

## شرح المثال

`sameorigin` يسمح بإطار من نفس الأصل فقط، وهو قريب من الافتراضي `SAMEORIGIN`. سياسة الإحالة تمنع إرسال مسار كامل إلى أصل آخر. لا تطفئ CSP في الإنتاج لتسريع Swagger على HTTP البعيد. الصفحة تقترح HTTPS أو إزالة `upgrade-insecure-requests` في ذلك الظرف فقط.

رأس مسار واحد يمكن تجاوزه بـ `@Header("Cross-Origin-Resource-Policy", "cross-origin")` لصورة يجب أن تُحمَّل في موقع آخر.

## ماذا يحدث داخليًا؟

الوسيط أو خطاف `onRequest` يكتب الرؤوس مبكرًا. `enableCsrfProtection` يشارك الخطاف، والرؤوس تُكتب حتى على الطلب المرفوض. متصفح زار الموقع مرة على HTTPS يتذكر HSTS حسب `max-age` فيرفض HTTP لاحقًا. على HTTP المحلي قد يتجاهل المتصفح HSTS، وهذا متوقع.

CSP لا يوقف حقن SQL. يوقف تحميل سكربت من أصل لم تسمح به حين تُعرض صفحة.

## مثال عملي

بعد التفعيل، افحص استجابة `GET /posts` في الشبكة: `X-Content-Type-Options: nosniff` و `X-Frame-Options`. افتح Swagger على `/api` محليًا وتأكد أن الواجهة تظهر. إن كسرتها CSP بعد تخصيص خاطئ، راجع `scriptSrc` قبل تعطيل السياسة كلها. لا تضع `contentSecurityPolicy: false` في الإنتاج كحل دائم.

## أخطاء شائعة

- خيار `hsts` أو `frameguard` على `useSecurityHeaders`.
- `'unsafe-inline'` في سكربت الإنتاج.
- نداء الدالة بعد `listen`.
- ظن أن الرؤوس تحمي `curl` من سرقة JWT. هي تعليمات متصفح.

## متى نستخدمه؟

على كل تطبيق يخدم متصفحًا أو Swagger أو HTML، منذ بداية الإنتاج.

## متى لا نستخدمه؟

كبديل عن تصديق المدخلات. الرأس لا يفحص `title` في JSON.

## مقارنة

| `useSecurityHeaders` | `helmet()` مباشرة |
| --- | --- |
| مدمج منذ 12.1 | حزمة على المنصة |
| أسماء حديثة | المقالات القديمة قد تستخدم أسماءً مرفوضة هنا |
| بلا دالة nonce لكل طلب | Helmet يقبل دوالًا في CSP |

## تمارين

### تمرين سهل

أي استدعاء يفعّل الرؤوس المدمجة؟

<details>
<summary>الحل</summary>

`app.useSecurityHeaders()` قبل `listen`.

</details>

### تمرين متوسط

لماذا يُرفض خيار اسمه `hsts`؟

<details>
<summary>الحل</summary>

اسم قديم في Helmet. الواجهة الحالية تتوقع `strictTransportSecurity`.

</details>

### تمرين متقدم

Swagger يعمل محليًا ويتكسر بعد تشديد `script-src` بحذف `'self'`. ماذا تعيد؟

<details>
<summary>الحل</summary>

`'self'` لأن واجهة Swagger تُحمَّل من تطبيقك. لا تفتح `'unsafe-inline'` للإنتاج إن كان الافتراضي يكفي.

</details>

## أسئلة مراجعة

- ماذا يُحذف من الرؤوس افتراضيًا؟
- هل `X-XSS-Protection: 0` يعني إهمال XSS؟
- متى تُستدعى الدالة نسبة إلى `listen`؟

## الخلاصة

`useSecurityHeaders` يضع رؤوس Helmet 8 على المنصتين. خصّص بالأسماء الحالية، وأبقِ CSP ضيقة، ولا تعتبر الرؤوس فحصًا للمدخلات.

## ماذا بعد؟

[Rate Limiting و @nestjs/throttler](./03-rate-limiting.md) يبطئ التخمين والضغط.

## المصادر الرسمية

- NestJS — Security headers: https://docs.nestjs.com/security/helmet
