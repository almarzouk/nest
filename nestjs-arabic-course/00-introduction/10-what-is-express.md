# ما هو Express

## الهدف من هذا الدرس

قراءة أصغر تطبيق Express كما في توثيقه، ومعرفة ماذا أضاف فوق `node:http`.

## المشكلة التي يحلها

مثال `createServer` في Node يرد الرد نفسه على كل طلب. تطبيق حقيقي يحتاج:

- مسارًا مختلفًا لكل مورد.
- طريقة HTTP مختلفة.
- مكانًا يضيف رأسًا أو يقرأ الجسم قبل الدالة النهائية.
- دوالًا صغيرة تتراكم بدل `if` عملاق على `req.url`.

Express يعطي هذا التوجيه مع بقاء `req` و `res` قريبين من Node.

## الفكرة ببساطة

Express إطار ويب خفيف لـ Node.js. تنشئ تطبيقًا، تسجّل مسارات، وتستدعي `listen`. هو يطابق الطلب الوارد إلى الدالة التي سجّلتها.

توثيق Nest يسميه minimalist web framework، مجرّبًا في الإنتاج، وبمجموعة كبيرة من الـ middleware المتوافقة. Nest تستخدمه افتراضيًا عبر الحزمة `@nestjs/platform-express`.

## التعريف التقني

تطبيق Express دالة/كائن يمثّل سلسلة معالجة. كل قطعة في السلسلة middleware أو معالج مسار. القطعة تقرأ الطلب، وقد تعدّل الرد، ثم إما تنهي الرد أو تمرر التحكم إلى القطعة التالية.

التوثيق الرسمي لمثال Hello World يقول صراحة: `req` و `res` هما الكائنان اللذان يعطيهما Node، وتقدر أن تستخدم `req.pipe()` و `req.on('data', ...)` كما تفعل من غير Express.

## Mental Model

```text
طلب وارد
   |
   v
سلسلة Express
   |-- middleware عام (مثل تحليل JSON) إن سجّلته
   |-- مسارات app.get / app.post / ...
   |-- إن لم يطابق شيء: 404 من Express
   v
الاستجابة على res
```

## أبسط مثال

من توثيق Express، نسخة TypeScript لأنها الأقرب لما ستكتبه مع Nest:

```ts
import express, { type Express, type Request, type Response } from 'express';

const app: Express = express();
const port = 3000;

app.get('/', (req: Request, res: Response) => {
  res.send('Hello World!');
});

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`);
});
```

التوثيق يعرض أيضًا `index.cjs` بـ `require` و `index.mjs` بـ `import`. المعنى واحد. مشاريع NestJS 12 الجديدة تسأل عند الإنشاء: ESM هو الافتراضي ويستخدم Vitest، أو CommonJS ويستخدم Jest. ستختار بوعي في الفصل `02`. المثال هنا ESM لأنه الشكل الذي يعرضه توثيق Express الحديث إلى جانب CommonJS.

تشغيل المثال، كما في التوثيق: مجلد، ثم `npm init`، ثم تثبيت `express`، ثم `node` على الملف الناتج. مع TypeScript تحتاج ترجمة أو مشغّلًا. لا تبنِ مشروع Nest في هذا الدرس.

بعد التشغيل، المتصفح على `http://localhost:3000/` يرى `Hello World!`. أي مسار آخر يأخذ `404 Not Found` من Express. هذه جملة التوثيق، لا إضافة منا.

## شرح المثال

- `import express from 'express'` يستورد دالة المصنع. استدعاؤها يبني التطبيق.
- `Express` و `Request` و `Response` أنواع TypeScript فقط. `import type` و `type` داخل القوس يعني أنها تُمحى عند الترجمة ولا توجد وقت التشغيل.
- `const app = express()` التطبيق. ليس خادمًا يستمع بعد. هو مكان تسجيل المسارات.
- `port` رقم المنفذ الذي ستمرره إلى `listen`.
- `app.get('/', handler)` يسجّل معالجًا لطريقة `GET` على المسار `/` بالضبط. أنت لا تنادي `handler` هنا.
- `req` الطلب. غير مستخدم لأن الرد ثابت. وجوده في المعاملات لأن Express ينادي المعالج بهذا الشكل دائمًا: `(req, res, next)` و `next` اختياري في الكتابة إذا لن تمرر التحكم.
- `res.send('Hello World!')` يكتب الجسم وينهي الاستجابة. Express يضبط بعض الرؤوس، منها نوع المحتوى المناسب للنص. هي راحة فوق `res.end`.
- `app.listen(port, callback)` يبدأ الاستماع. تحت الغطاء Express ينشئ خادم `node:http` ويناديه بـ `listen`. الـ callback تعمل بعد نجاح الاستماع، مثل مثال Node.
- القوس في `console.log` قالب نص يدرج `port`.

## ماذا يحدث داخليًا؟

1. `express()` يبني تطبيقًا فيه مكدس طبقات (stack) فارغ تقريبًا من مساراتك، مع سلوك افتراضي للإطار.
2. `app.get` يضيف طبقة تعرف الطريقة `GET` والمسار `/`.
3. `listen` يحجز المنفذ عبر خادم HTTP الخاص بـ Node.
4. عند طلب، Express يمر على الطبقات بالترتيب.
5. الطبقة التي تطابق الطريقة والمسار تُستدعى.
6. `res.send` ينهي الرد.
7. إذا لم تطابق طبقة، Express ينهي الرد بـ `404`.

لا يوجد في هذا المثال تحليل JSON، ولا مسارات بمعاملات، ولا فصل ملفات. الإطار لا يفرض مجلد `controllers`. هذه حرية، وهي نفسها أصل الفوضى عندما يكبر الفريق.

## مثال عملي

كتابان في الذاكرة، لنرى أن Express ما زال قريبًا من HTTP:

```ts
import express, { type Express, type Request, type Response } from 'express';

const app: Express = express();

const books = [
  { id: '1', title: 'Clean Code' },
  { id: '2', title: 'Refactoring' },
];

app.get('/books', (_req: Request, res: Response) => {
  res.json(books);
});

app.get('/books/:id', (req: Request, res: Response) => {
  const book = books.find((item) => item.id === req.params.id);
  if (!book) {
    res.status(404).json({ message: 'Book not found' });
    return;
  }
  res.json(book);
});

app.listen(3000);
```

- `res.json` يرسل JSON ويضبط `Content-Type` المناسب. هي `stringify` مضافًا إليها إنهاء الرد.
- `:id` قطعة مسار يلتقطها Express ويضع النص في `req.params.id`. في REST هذا معرّف المورد.
- `_req` تسمية تعني أن المعامل غير مستخدم. المعالج الثاني يستخدم `req`.
- `return` بعد `404` يمنع إرسال رد ثانٍ. إرسال ردين على الاتصال نفسه يسبب خطأ.
- البيانات في مصفوفة داخل الملف. هذا مقبول لمثال، وغير مقبول كتصميم: إعادة تشغيل العملية تمسح الكتب، وكل المسارات ترى المصفوفة مباشرة.

المقابل الذي ستكتبه لاحقًا في Nest، من غير شرح زائد:

```ts
@Get()
findAll() {
  return books;
}

@Get(':id')
findOne() {
  return book;
}
```

الفرق الظاهر: Nest تستطيع إرجاع القيمة، والإطار يكتب الاستجابة. في Express أنت تنادي `res.json` بنفسك. اسم هذا الفرق في توثيق Nest: Standard response handling مقابل التحكم المباشر بكائن الرد. سنفصله في Controllers. المعاينة هنا حتى تربط ما تعرفه بما سيأتي.

## أخطاء شائعة

- نسيان `app.listen` ثم استغراب أن المنفذ مغلق. التسجيل وحده لا يفتح مقبسًا.
- استدعاء `res.send` مرتين في المسار نفسه.
- توقع أن `req.body` موجود في مثال Hello World. تحليل JSON طبقة تسجّلها أنت، غالبًا `express.json()`، وليست سحرًا في كل تطبيق. سنضعها في الدرس التالي.
- كتابة كل التطبيق في معالج واحد فيه `if (req.url === ...)`. هذا رجوع إلى Node الخام مع خطوات إضافية.

## متى نستخدم Express مباشرة؟

خدمة صغيرة، أو طبقة رقيقة، أو عندما تريد قرارًا كاملًا على شكل الملفات. كثير من الإنتاج يعمل على Express منذ سنوات. Nest لا تلغي هذا؛ التوثيق يقول إن Nest تعرض لك APIs الخاصة بالمنصة عندما تحتاجها.

## متى ننتقل إلى طبقة ذات رأي؟

عندما يتكرر عندك: أين أضع القاعدة، كيف أختبر مسارًا من غير تشغيل الشبكة كلها، كيف أمنع مسارًا من نسيان فحص الصلاحية، وكيف يعرف مطور جديد أين يضيف موردًا. الدرس 12 يجمع هذه الآلام. لا تنتقل لأن Express "قديم". انتقل لأن هيكل الفريق صار المشكلة.

## مقارنة مع node:http

| | `node:http` | Express |
| --- | --- | --- |
| مطابقة Method والمسار | تكتبها | `app.get` و `app.post` |
| معاملات المسار | تكتبها | `:id` |
| إرسال JSON | `JSON.stringify` و `end` | `res.json` |
| كائن الطلب | IncomingMessage | نفسه مع إضافات Express |
| هيكل تطبيق | لا يقترح | لا يقترح مجلدات Nest |

## تمارين

### تمرين سهل

في مثال Hello World، ماذا يحدث لطلب `POST /` وطلب `GET /books`؟

<details>
<summary>الحل</summary>

كلاهما `404`. المسجّل فقط `GET` على `/`. الطريقة الأخرى والمسار الآخر لا يطابقان الطبقة.

</details>

### تمرين متوسط

أضف إلى مثال الكتب سلوك REST الناقص: `POST /books` غير موجود. صف، من غير بناء Nest، ماذا تحتاج تسجيله حتى يُنشأ كتاب من جسم `{ "title": "..." }` ويُعاد `201`. اذكر الطبقة التي تنقص مثالنا لقراءة JSON.

<details>
<summary>الحل</summary>

تسجيل `app.post('/books', handler)` يقرأ العنوان، يضيف كتابًا بمعرّف يولّده الخادم، ويرسل `res.status(201).json(book)`. قبل ذلك نحتاج `app.use(express.json())` حتى يُملأ `req.body`. من غير هذه الطبقة يبقى الجسم بايتات غير محللة.

</details>

### تمرين متقدم

لماذا يستطيع مسار Express أن يستدعي `req.on('data', ...)` رغم أنك لم تنشئ الخادم بـ `createServer` بنفسك؟

<details>
<summary>الحل</summary>

لأن `req` هو IncomingMessage القادم من Node. `app.listen` ينشئ خادم HTTP تحته، و Express يمرر كائنات Node إلى الطبقات مع توسيعها. التوثيق يذكر هذا حتى لا تظن أن Express استبدل بروتوكولًا آخر.

</details>

## أسئلة مراجعة

- ماذا يسجّل `app.get`، ومتى تُستدعى الدالة؟
- ما الفرق بين إنشاء `app` واستدعاء `listen`؟
- ماذا يعيد Express لمسار غير مسجّل؟
- ما العلاقة بين `req` في Express و `node:http`؟

## الخلاصة

Express تطبيق تسجّل عليه المسارات، وهو يطابق الطلب ويستدعي دالتك، و `req` و `res` ما زالا كائني Node. الإطار رقيق: يحل التوجيه، ويترك هيكل المنتج قرارًا بشريًا.

## ماذا بعد؟

المثال أخفى السلسلة التي تمر قبل المعالج النهائي. الدرس التالي يفتحها: Middleware، و `next`، ولماذا الترتيب يغيّر السلوك.

## المصادر الرسمية

- Express — Hello world: https://expressjs.com/en/starter/hello-world.html
- Express — Routing: https://expressjs.com/en/guide/routing.html
- Express — Writing middleware: https://expressjs.com/en/guide/writing-middleware.html
- NestJS — First steps (منصة Express الافتراضية): https://docs.nestjs.com/first-steps
- NestJS — Performance (Fastify) للمقارنة لاحقًا في هذا الفصل: https://docs.nestjs.com/http/performance
