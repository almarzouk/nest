# مقارنته بـ Express Middleware

## الهدف من هذا الدرس

تبيين ما الذي يتطابق بين Middleware في Express و Middleware في Nest، وأين يختلف التسجيل والمنصة.

## المشكلة التي يحلها

من يأتي من Express يظن أن `app.use` هو المكان الوحيد، أو أن كائن `req` على Fastify هو نفسه كائن Fastify. الظنان يكسران نقل كود قديم.

## الفكرة ببساطة

العقد واحد مع محوّل Express: `(req, res, next)`. مكان التسجيل في Nest غالبًا `configure` على الوحدة، مع الإبقاء على `app.use` للدوال العامة التي لا تحتاج حقنًا.

## التعريف التقني

صفحة Middleware تقول إن Middleware في Nest يعادل Express افتراضيًا، وتنقل قدرات Express: تنفيذ كود، تعديل الطلب أو الرد، إنهاء الدورة، واستدعاء `next()`.

الفرق الذي تسميه الصفحة تحذيرًا: Express و Fastify يتعاملان مع Middleware بتوقيعات مختلفة. التفاصيل في فصل Fastify. لا تفترض أن `req` على Fastify هو `FastifyRequest`. الدعم هناك يمر بطبقة وسيطة، والتوقيع الذي يصلك قد يكون طلب Node الخام.

`app.use()` على تطبيق Nest يمرر الدالة إلى المنصة نفسها. التوثيق يفرّق: هذه الدالة خارج `MiddlewareModule`. الاستثناء الذي ترمية من Middleware مربوط عبر `MiddlewareConsumer` هو الذي تصل إليه طبقة استثناءات Nest بسهولة. من `app.use` فضّل رمي الخطأ أو `next(err)` بالطريقة التي تفهمها المنصة، واربط منطق Nest عبر `configure` حين تريد تلك الطبقة.

## Mental Model

```text
Express القديم
  app.use(helmet)
  app.use('/loans', stampShelf)
  app.get('/loans', handler)

Nest على Express
  app.use(stampShelf)          ← منصة، بلا حاوية DI
  LoansModule.configure        ← NestMiddleware / دالة، مع forRoutes
  handler داخل Controller
```

الدالة `stampShelf` يمكن أن تبقى كما كُتبت لـ Express. الذي يتغير عنوان السكن.

## أبسط مثال

```ts
import type { NextFunction, Request, Response } from 'express';

export function stampShelf(
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  res.setHeader('x-shelf', 'loans');
  next();
}
```

في Express:

```ts
app.use('/loans', stampShelf);
```

في Nest، الدالة نفسها تُمرَّر إلى `consumer.apply(stampShelf)` أو إلى `app.use(stampShelf)`. جسم الدالة لا يتغير.

## شرح المثال

- التوقيع `Request` و `Response` و `NextFunction` من حزمة `express` صحيح لمحوّل Express، وهو الافتراض في أمثلة التوثيق.
- `app.use` في `main.ts` يسجّل الدالة لكل المسارات التي تصل إلى المنصة، وبالترتيب الذي تنادي به `use`.
- `apply` داخل الوحدة يقيّد المسارات لاحقًا بـ `forRoutes`. هذا القيد أداة Nest فوق العقد نفسه.
- دالة بلا اعتماديات تبقى دالة. الصنف يظهر حين تحتاج `constructor`، وذلك درس الصنف.

## ماذا يحدث داخليًا؟

1. `NestFactory.create` يبني محوّل HTTP. الافتراض الشائع Express.
2. `app.use(fn)` يضيف `fn` إلى سلسلة المنصة قبل مسارات Nest.
3. `configure` يجمع دوالًا وأصنافًا ويسجّلها عبر `MiddlewareModule` على المسارات المطابقة.
4. عند الطلب تُنفَّذ سلسلة `app.use` أولًا، ثم Middleware الوحدات، كما في دورة الطلب.
5. على Fastify، الصفحة تحذّر من اختلاف التوقيع. انسخ دالة Express إلى مشروع Fastify بعد قراءة ذلك الفصل، لا قبلها.

حزم مثل التي تتوقع `(req, res, next)` تُستدعى داخل `apply` بالطريقة نفسها: `consumer.apply(someExpressMiddleware()).forRoutes(...)`. استدعِ المصنع إن كانت الحزمة تصدّر مصنعًا، ومرّر الناتج الذي هو دالة Middleware.

## مثال عملي

مشروع إعارات كان `server.js` فيه `app.use(stampShelf)` ثم مسارات كتب. نقل المسارات إلى `LoansController` يبقي `stampShelf` كما هي. في `main.ts` سطر `app.use(stampShelf)` يعيد السلوك العام. حين تريد الختم على `/loans` فقط، انقل الدالة إلى `configure` في درس لاحق واترك `app.use` للختم الذي يخص كل طلب.

## أخطاء شائعة

- إعادة كتابة دالة Express كصنف مع أن لها صفر اعتماديات. التوثيق يفضّل الدالة في هذه الحالة.
- تمرير صنف إلى `app.use`. التسجيل العام عبر `app.use` لا يدخل حاوية الحقن. الصنف الذي يحتاج مزوّدًا يُربَط من الوحدة.
- افتراض أن `req.query` على Fastify بنفس شكل Express لأن الملف اسمه `middleware`. اقرأ تحذير الصفحة.
- وضع `express.json()` مرة ثانية. المحوّل سجّله. التكرار يفسد الجسم أو يرمي. الإيقاف الصريح هو `bodyParser: false`.

## متى نستخدمه؟

حين تنقل دالة Middleware تعمل اليوم على Express، أو حين توثّق للفريق أن العقد لم يتغير وأن السكن هو الذي تغيّر.

## متى لا نستخدمه؟

حين يكون السؤال «كيف أقيّد المسار» أو «كيف أحقن خدمة». ذلك `forRoutes` والصنف. المقارنة هنا فقط بين العقدين.

## مقارنة

| | Express | Nest على Express |
| --- | --- | --- |
| التوقيع | `(req, res, next)` | التوقيع نفسه للدالة |
| التسجيل العام | `app.use` | `app.use` على `INestApplication` |
| التسجيل على مسار | `app.use('/loans', fn)` | `apply(fn).forRoutes(...)` |
| الحقن | تغلق على متغير خارجي | صنف Middleware داخل الوحدة |
| Fastify | غير معني | التوقيع قد يختلف، راجع فصل Fastify |

## تمارين

### تمرين سهل

هل تتغير معاملات `stampShelf` حين تنتقل من `app.use` في Express إلى `app.use` في Nest على المحوّل نفسه؟

<details>
<summary>الحل</summary>

لا. الدالة تبقى `(req, res, next)`. الذي يتغير مكان الاستدعاء.

</details>

### تمرين متوسط

لماذا لا تضع صنفًا يعتمد على `LoansService` داخل `app.use`؟

<details>
<summary>الحل</summary>

`app.use` تسجّله المنصة خارج حاوية DI. التوثيق يقول إن Middleware العام بهذه الطريقة لا يصل إلى الحاوية. الصنف يُربَط من `configure` في وحدة ترى الخدمة.

</details>

### تمرين متقدم

الفريق نقل `stampShelf` إلى Fastify لأن مقياس الأداء طلب ذلك، ثم صار `req.header` يرمي. ما أول صفحة تقرأها، وما الفرض الذي سقط؟

<details>
<summary>الحل</summary>

صفحة Middleware تحذّر أن Express و Fastify يختلفان في التوقيع، وتشير إلى فصل Fastify. الفرض الساقط أن `req` ما زال كائن Express. الدالة تُراجَع على التوقيع الفعلي للمحوّل قبل إعادة استخدامها.

</details>

## أسئلة مراجعة

- ما الذي يتطابق حرفيًا مع Express على محوّل Express؟
- أين يُسجَّل Middleware العام الذي لا يحتاج حقنًا؟
- لماذا تحذّر الصفحة عند ذكر Fastify؟
- من يلتقط استثناء Middleware المربوط بالمنصة مقارنة بالمربوط عبر الوحدة؟

## الخلاصة

العقد مع Express هو العقد نفسه. Nest يضيف سكنًا داخل الوحدة وحقنًا للأصناف، ويبقي `app.use` للدالة العامة. على Fastify التوقيع سؤال منفصل تجيب عنه صفحة المحوّل.

## ماذا بعد؟

الدرس التالي يكتب Middleware كدالة فقط: متى تكفي، وكيف تُمرَّر إلى `apply` من غير صنف.

## المصادر الرسمية

- [Middleware](https://docs.nestjs.com/middleware)
- [Request lifecycle](https://docs.nestjs.com/faq/request-lifecycle)
