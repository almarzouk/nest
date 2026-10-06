# كيف يعمل Express

## الهدف من هذا الدرس

تتبع طلب واحد داخل مكدس Express حتى المعالج النهائي، لأن Nest تعيد استخدام الفكرة نفسها في طبقة Middleware.

## المشكلة التي يحلها

التطبيق الذي يضع كل شيء داخل `app.get` يصعب فيه مشاركة سلوك: تسجيل الطلب، وتحليل JSON، ورفض من لا يحمل مفتاحًا. نريد دوالًا تمر على الطلب بالترتيب، وكل واحدة تكمل أو توقف السلسلة.

## الفكرة ببساطة

Express يمرر الطلب عبر طبقات بالترتيب الذي سجّلت به. الطبقة دالة بشكل:

```ts
(req, res, next) => { /* ... */ }
```

- إذا أنهت الرد عبر `res.send` أو `res.json` أو `res.end`، فالسلسلة تتوقف عمليًا لأن الرد اكتمل.
- إذا نادت `next()`، فالتحكم ينتقل إلى الطبقة التالية.
- إذا نادت `next(error)`، فالتحكم يقفز إلى معالج أخطاء.

Middleware اسم هذه الطبقة عندما تكون في وسط الطريق، لا بالضرورة "النهائية" التي تعرف المورد.

## التعريف التقني

Middleware دالة لها وصول إلى كائن الطلب وكائن الرد والدالة `next` في دورة الطلب والاستجابة. قد تنفّذ كودًا، أو تغيّر `req` و `res`، أو تنهي الدورة، أو تستدعي `next` لتمرر التحكم.

هذا تعريف دليل Express عن كتابة الـ middleware، بصياغة عربية هنا.

Nest تعرّف Middleware بجملة قريبة: دالة تُستدعى قبل route handler. عندما نصل إلى فصل Middleware في Nest سترى `NestMiddleware` و `configure`. الترتيب الذهني الذي تبنيه الآن هو نفسه: قبل المعالج، طبقة بعد طبقة.

## Mental Model

```text
app.use(log)
app.use(express.json())
app.get('/books', listBooks)

GET /books
   |
   v
 log -- next() --> json parser -- next() --> listBooks -- res.json
```

ترتيب `use` و `get` هو ترتيب المرور للطلب الذي يطابقها.

## أبسط مثال

```ts
import express, { type Express, type NextFunction, type Request, type Response } from 'express';

const app: Express = express();

app.use((req: Request, res: Response, next: NextFunction) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

app.get('/books', (_req: Request, res: Response) => {
  res.json([{ id: '1', title: 'Clean Code' }]);
});

app.listen(3000);
```

## شرح المثال

- `app.use(fn)` يسجّل طبقة لكل الطرق والمسارات، لأننا لم نمرر مسارًا. ستعمل قبل المسارات المسجّلة بعدها.
- المعاملات الثلاثة هي عقد Express: الطلب، الرد، والتمرير.
- `req.method` مثل `GET`. `req.url` المسار مع query إن وُجدت.
- `console.log` أثر جانبي على الخادم. لا يغيّر الرد.
- `next()` يسلم إلى الطبقة التالية. من غير هذا الاستدعاء، ومن غير إنهاء `res`، يبقى العميل منتظرًا حتى تنتهي المهلة. هذا من أكثر أعطال Express صمتًا.
- `app.get` طبقة لاحقة. لا تُستدعى إلا إذا وصلت إليها السلسلة وطابقت الطريقة والمسار.
- `res.json` ينهي الرد. لا حاجة إلى `next()` بعد الإنهاء.

طلب `GET /books` يطبع السطر ثم يعيد المصفوفة. طلب `POST /books` يطبع السطر ثم يأخذ `404` لأن لا طبقة `POST`، والطباعة حصلت قبل قرار 404.

## ماذا يحدث داخليًا؟

Express يحتفظ بمكدس. لكل طلب ينشئ مؤشرًا على الطبقة الحالية.

1. يدخل الطبقة الأولى.
2. إن استُدعيت `next()` من غير وسيط، يزيد المؤشر وينادي الطبقة التالية المتطابقة.
3. إن كُتب الرد وانتهت الدالة، لا توجد طبقة تالية تُستدعى إلا إذا استدعاها الكود بالخطأ بعد الإنهاء.
4. `express.json()` طبقة جاهزة: تقرأ الجسم إذا كان `Content-Type` يدل على JSON، ثم تضع الناتج في `req.body`، ثم تنادي `next()`. إن كان JSON مكسورًا تمرر خطأ.
5. معالج المسار طبقة تتطابق بطريقة ومسار. `:id` قاعدة ضمن المسار، والنص المطابق يذهب إلى `req.params`.

`express.json()` ليست مفعّلة لأنك ثبّت Express. هي middleware تختارها. موضعها في المكدس مهم: مسار قبلها يرى `req.body` غير محللة.

معالج الأخطاء طبقة بأربعة معاملات:

```ts
app.use((err: unknown, _req: Request, res: Response, _next: NextFunction) => {
  res.status(500).json({ message: 'Unexpected error' });
});
```

Express يميّزه بعدد المعاملات. يُسجَّل بعد المسارات. `next(err)` من طبقة سابقة يقفز إليه ويتجاوز المسارات العادية. لن نبني نظام أخطاء Express كاملًا. في Nest، Exception Filter يحتل هذا الدور على مستوى الإطار، مع فرق مهم في الترتيب ستقرأه في فصل Request Lifecycle: الفلتر في Nest يلتقط ما لم يُمسك أثناء معالجة الطلب، وترتيب اختياره من الأخص إلى الأعم.

نقطة من توثيق Nest عن Fastify، للعلم فقط: عند استخدام Fastify، دوال middleware تستقبل `req` و `res` الخام لا أغلفة Fastify، بسبب الحزمة التي يستخدمها Nest. لا تخلط هذا مع Express. على منصة Express، كائنات Express هي التي تعرفها.

## مثال عملي

تحليل JSON ثم رفض جسم فارغ، ثم إنشاء كتاب. البيانات في الذاكرة للتوضيح.

```ts
import express, { type NextFunction, type Request, type Response } from 'express';

const app = express();
const books: { id: string; title: string }[] = [];

app.use(express.json());

app.use((req: Request, res: Response, next: NextFunction) => {
  if (req.method === 'POST' && req.path === '/books') {
    const title = req.body?.title;
    if (typeof title !== 'string' || title.length === 0) {
      res.status(400).json({ message: 'title is required' });
      return;
    }
  }
  next();
});

app.post('/books', (req: Request, res: Response) => {
  const book = { id: String(books.length + 1), title: req.body.title as string };
  books.push(book);
  res.status(201).json(book);
});

app.listen(3000);
```

المسار:

```text
POST /books  { "title": "" }
   -> express.json يملأ req.body
   -> الطبقة الثانية ترى عنوانًا فارغًا وترسل 400 ولا تنادي next
   -> app.post لا يعمل

POST /books  { "title": "Refactoring" }
   -> json
   -> الطبقة الثانية تنادي next
   -> app.post يحفظ ويرسل 201
```

الطبقة الثانية بداية لفكرة Pipe: افحص المدخل قبل الدالة. في Express مكان الفحص قرارك، وقد تنسى ربطه على `POST /reviews`. في Nest ستضع القاعدة على DTO فتسري حيث يُستخدم الـ DTO. الفرق معماري، والكود أعلاه يكفي لرؤية المشكلة.

## أخطاء شائعة

- نسيان `next()` مع عدم إنهاء الرد. العميل يعلّق.
- استدعاء `next()` وبعدها `res.json` أيضًا. رد مزدوج أو استجابة بعد أن بدأت طبقة أخرى بالكتابة.
- وضع `express.json()` بعد المسار الذي يقرأ `req.body`. الترتيب هو التنفيذ.
- حشر الصلاحية وقواعد العمل والتسجيل في middleware واحد عام. الطبقة تصبح تطبيقًا خفيًا. كل مسؤولية تستحق طبقة أو مكانًا أوضح عندما نصل إلى Guards و Pipes و Interceptors.
- الاعتماد على `req.body.title as string` لإسكات TypeScript. هذا لا يفحص القيمة. الفحص هو `typeof` في المثال، أو Validation لاحقًا.

## متى نستخدم Middleware في Express؟

لسلوك عرضي على الطلب قبل معرفة تفاصيل المورد: تسجيل، تحليل جسم، رأس أمني عام، قياس زمن. ولأي شيء يجب أن يمر قبل اختيار المعالج.

## متى لا نستخدمه؟

لقاعدة تخص موردًا واحدًا وشكل جسمه، إذا كان وضعها في middleware عام يخفي القاعدة عن قارئ المسار. ضعها قرب المسار أو في مكان الإطارات المخصصة للتحقق. المقارنة الكاملة بين Middleware و Guard و Interceptor و Pipe فصلها الخاص بعد أن تُبنى الأربعة. الجدول التالي خريطة أسماء فقط:

| الأداة لاحقًا في Nest | قريبها الذهني الآن |
| --- | --- |
| Middleware | طبقة Express قبل اختيار السلوك الدقيق للمسار |
| Guard | هل يُسمح لهذا الطلب أن يكمل |
| Pipe | حوّل المدخل أو ارفضه |
| Interceptor | لفّ حول تنفيذ المعالج قبلًا وبعدًا |
| Exception Filter | حوّل الاستثناء غير الممسوك إلى رد |

لا تستخدم الجدول كمرجع تنفيذ. هو سبب وجود فصول منفصلة: Express يضع أكثر هذه الأدوار في شكل دالة `(req, res, next)` واحدة، والفرق يصبح اتفاق فريق. Nest يفصل الأماكن.

## مقارنة مع Node الخام

على `node:http` تكتب بنفسك:

```ts
if (req.method === 'GET' && req.url === '/books') {
  // ...
}
```

Express يستبدل سلسلة `if` بمكدس طبقات ومطابقة مسارات. أنت ما زلت تصمم ترتيب الطبقات.

Nest ستأخذ المكدس نفسه على منصة Express: Middleware الذي تعرّفه في Nest يُربَط على Express تحت الغطاء. لهذا يفيدك فهم `next`. لن تكتب `next` في Controller العادي. ستكتبه في Middleware عندما نصل إليه.

## تمارين

### تمرين سهل

طبقة تطبع `A` ثم `next()`، ثم طبقة تطبع `B` ثم `next()`، ثم `GET /` يرسل `ok`. ماذا يُطبع لطلب `GET /`؟

<details>
<summary>الحل</summary>

`A` ثم `B` ثم يُرسل `ok`. الترتيب هو ترتيب التسجيل.

</details>

### تمرين متوسط

الطبقة الأولى ترسل `res.status(401).json({ message: 'no' })` ولا تنادي `next`. هل يعمل المعالج النهائي؟ ماذا يرى العميل؟

<details>
<summary>الحل</summary>

المعالج النهائي لا يعمل لأن السلسلة لم تُمرَّر إليه. العميل يرى `401` والجسم `{ "message": "no" }`.

</details>

### تمرين متقدم

لماذا فحص `title` في المثال العملي ما زال هشًا حتى بعد أن كتبنا middleware؟ اذكر سيناريو يمر فيه عنوان غير نصي إلى `app.post`، أو سيناريو لا يعمل فيه الفحص لعملية إنشاء أخرى.

<details>
<summary>الحل</summary>

الفحص مربوط بشرط `req.path === '/books'` فقط. `POST /notes` لا يُفحص. كذلك الشرط يعتمد على أن `express.json` سبقه؛ لو انعكس الترتيب فـ `req.body` غير جاهز وقد يرمي الفحص أو يرفض كل شيء. و `req.body.title as string` في المعالج يوثق نية لا يفرضها المترجم على الشبكة. مسار جديد يحتاج تذكر الشرط. هذا نوع التكرار الذي ستنقله Pipes إلى شكل البيانات نفسه.

</details>

## أسئلة مراجعة

- ما الأشكال الثلاثة التي تنهي فيها طبقة عملها: رد، أم `next`، أم خطأ؟
- لماذا يعلّق المتصفح إذا لم يحدث أي منها؟
- لماذا ترتيب `express.json()` بالنسبة للمسار مهم؟
- ماذا ستقابل `next` لاحقًا داخل Nest، وأين لن تحتاج كتابتها؟

## الخلاصة

Express مكدس دوال بالترتيب. `next()` يمرر التحكم، وإنهاء `res` يرسل الرد، و `next(error)` يبحث عن معالج أخطاء. Nest على Express تبني فوق هذا المكدس وتضيف أماكن أوضح لبقية المسؤوليات.

## ماذا بعد؟

الآلية تكفي لمشروع صغير. الدرس التالي ينظر إلى المشروع بعد سنة: أين تنكسر هذه الحرية، ولماذا صارت المعمارية هي المشكلة لا توجيه المسار.

## المصادر الرسمية

- Express — Writing middleware: https://expressjs.com/en/guide/writing-middleware.html
- Express — Routing: https://expressjs.com/en/guide/routing.html
- Express — Error handling: https://expressjs.com/en/guide/error-handling.html
- NestJS — Middleware: https://docs.nestjs.com/middleware
- NestJS — Request lifecycle: https://docs.nestjs.com/faq/request-lifecycle
