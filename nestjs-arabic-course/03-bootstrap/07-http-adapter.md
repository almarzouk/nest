# HTTP Adapter

## الهدف من هذا الدرس

معرفة الطبقة التي تترجم مسارات Nest إلى Express أو Fastify أثناء `create`.

## المشكلة التي يحلها

لو سجل كل Controller نداء `app.get` بنفسه، لصارت الخدمات تعرف Express. المحوّل يبقي التسجيل في Nest ويمرره إلى المنصة.

## الفكرة ببساطة

HTTP Adapter جسر. Nest تقول "اربط `GET /` بهذه الدالة"، والمحوّل ينفذ ذلك على المنصة الحالية.

## التعريف التقني

فصل Performance: الاستقلال عن الإطار يتم عبر adapter يمرر middleware والمعالجات إلى تنفيذ المكتبة. الافتراضي `@nestjs/platform-express`. Fastify عبر `FastifyAdapter` من `@nestjs/platform-fastify`. سؤال التوثيق HTTP adapter يشرح الوصول إلى الخادم الخام عندما تحتاجه مكتبة خارجية.

## Mental Model

```text
@Controller / @Get
      |
      v
HTTP Adapter
      |
      +-- Express app.get(...)
      +-- أو Fastify
```

## أبسط مثال

```ts
const app = await NestFactory.create(AppModule);
```

لا ترى المحوّل لأنه الافتراضي. هو موجود داخل `create`.

## شرح المثال

- عدم تمرير وسيط ثانٍ يعني Express.
- مسارات الزينة تُترجم أثناء البناء لا عند كل طلب كإعادة تسجيل.
- `listen` على كائن Nest يستدعي استماع المنصة عبر المحوّل.

## ماذا يحدث داخليًا؟

عند الإقلاع يمشي المحوّل على المتحكمات المسجّلة ويسجل كل طريقة ومسار. Middleware الذي تعرفه Nest يُربط على مكدس المنصة. لذلك فهم `next` في Express ما زال يفيد على المنصة الافتراضية. Guard و Pipe ليسا طبقات Express تعيد كتابتها؛ هما محطات Nest تُستدعى من الغلاف الذي سجله المحوّل حول المعالج.

الوصول إلى Express التحتي أداة حافة. استخدمها لتركيب مكتبة لا تعرف Nest، لا لقراءة رؤوس داخل Service.

## مثال عملي

مكتبة تقيس الطلبات وتتوقع تطبيق Express: تُركَّب مرة بعد `create` عبر واجهة المنصة، ثم تعمل لكل الطلبات. الخدمة تبقى بلا `import express`.

## أخطاء شائعة

- استيراد `express` داخل Service.
- افتراض أن وصفة `app.use` من مقال Express تعمل حرفيًا على Fastify.
- إنشاء تطبيق Express ثانٍ.

## متى تلمس المحوّل؟

عند مكتبة منصة، أو عند اختبار يحتاج `getHttpServer`، أو عند اختيار Fastify عن قياس لا عن تقليد.

## متى تتركه؟

في كل مسار عمل عادي. Controller يعيد قيمة ويكفي.

## مقارنة

| المنصة | كيف تُختار في create |
| --- | --- |
| Express | الافتراضي بلا وسيط |
| Fastify | `new FastifyAdapter()` |

## تمارين

### تمرين سهل

ما المنصة إن لم تمرر محوّلًا؟

<details><summary>الحل</summary>

Express.

</details>

### تمرين متوسط

متى تُسجل مسارات `@Get` على Express: عند الإقلاع أم عند الطلب؟

<details><summary>الحل</summary>

عند الإقلاع أثناء `create`.

</details>

### تمرين متقدم

لماذا إبقاء Service جاهلة باسم المنصة يحقق وعد platform-agnostic؟

<details><summary>الحل</summary>

لأن تبديل المحوّل يغيّر HTTP من غير إعادة كتابة القاعدة. إن استوردت الخدمة أنواع Express صارت جزءًا من المنصة لا من المجال.

</details>

## أسئلة مراجعة

- ماذا يترجم المحوّل؟
- أين يركب Fastify؟
- لماذا لا يُستورد Express في الخدمة؟

## الخلاصة

المحوّل يربط مسارات Nest بمنصة HTTP أثناء البناء. الافتراضي Express. عملك اليومي لا ينادي المحوّل.

## ماذا بعد؟

`listen` يفتح المنفذ على هذه المنصة.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — Performance (Fastify): https://docs.nestjs.com/http/performance
- NestJS — HTTP adapter: https://docs.nestjs.com/faq/http-adapter
