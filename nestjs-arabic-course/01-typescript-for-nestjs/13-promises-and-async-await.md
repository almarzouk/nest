# Promises و async/await

## الهدف من هذا الدرس

كتابة دوال Nest التي تنتظر قاعدة بيانات أو شبكة من غير حجب خيط JavaScript، وقراءة `Promise` في التوقيعات.

## المشكلة التي يحلها

قراءة كتاب من قاعدة بيانات لا تنتهي في اللحظة نفسها. إذا نام الخيط حتى ترجع النتيجة، توقفت بقية الطلبات، كما في درس Node. نحتاج تمثيلًا لنتيجة ستصل لاحقًا، وطريقة لقراءة الكود كأنه متتابع.

## الفكرة ببساطة

`Promise` قيمة ستنجح أو تفشل لاحقًا. `async` تجعل الدالة ترجع Promise. `await` يوقف تلك الدالة فقط حتى تستقر الـ Promise، ويحرر الخيط أثناء الانتظار.

## التعريف التقني

Promise لها حالات: معلّقة، ثم موفّاة بقيمة، أو مرفوضة بسبب. `await` على المرفوضة يرمي داخل الدالة غير المتزامنة، فيصير الرفض قابلًا لـ `try/catch`. Nest تنتظر Promise التي يرجعها المتحكم قبل إرسال الرد. استثناء غير ممسوك من هذه السلسلة يدخل Exception Filters.

## Mental Model

```text
find() تصل إلى await db
      |
      +-- الدالة تتوقف هي فقط
      +-- الخيط يخدم طلبًا آخر
      |
النتيجة تصل
      |
      v
find() تكمل بعد await
      |
      v
Nest ترسل الرد
```

## أبسط مثال

```ts
function loadTitle(): Promise<string> {
  return Promise.resolve('Refactoring');
}

async function label(): Promise<string> {
  const title = await loadTitle();
  return `Book: ${title}`;
}
```

## شرح المثال

- `loadTitle` ترجع Promise موفّاة فورًا. في الواقع سترجعها مكتبة قاعدة البيانات بعد الشبكة.
- `async` على `label` يجعل الإرجاع `Promise<string>` حتى لو كتبت `return` لنص عادي.
- `await` يستخرج `string` من `Promise<string>` داخل الدالة. خارجها ما زلت تحتاج `await label()` أو `.then`.
- رمي خطأ قبل الإرجاع، أو رفض الـ Promise، يجعل `label()` ترجع Promise مرفوضة.

## ماذا يحدث داخليًا؟

`async/await` تحويل إلى سلاسل Promise. الانتظار يسجّل تكملة الدالة لتعمل عندما تُحل الـ Promise، عبر حلقة أحداث Node. لا خيط جديد لكل `await`.

نسيان `await`:

```ts
async function bad(): Promise<string> {
  const pending = loadTitle();
  return `Book: ${pending}`;
}
```

`pending` هنا Promise لا نص، والنص يصبح شبيها بـ `Book: [object Promise]`. المترجم في `strict` مع فحص مناسب يحذّر غالبًا إن أُعد المشروع لذلك. عوّد نفسك على انتظار ما ترجع Promise إذا احتجت القيمة.

`void promise` تعني عمدًا أنك لا تنتظر. استخدمه نادرًا، وتأكد أن الرفض ممسوك، وإلا صار رفضًا غير معالج.

Nest: دالة متحكم `async` ترجع `Promise<Book>`. الإطار يفلّحها. لا تستدعِ `res.json` بنفسك إلا في الأسلوب الخاص بالمنصة.

## مثال عملي

```ts
async function requireBook(id: string): Promise<{ id: string; title: string }> {
  const book = await findBook(id);
  if (!book) {
    throw new Error(`Book ${id} not found`);
  }
  return book;
}

async function findBook(id: string): Promise<{ id: string; title: string } | null> {
  if (id === '2') return { id: '2', title: 'Refactoring' };
  return null;
}
```

`throw` داخل `async` يرفض الـ Promise. مستدعٍ بلا `try/catch` يمرر الرفض للأعلى. في Nest سترمي استثناء HTTP لا `Error` عامًا عندما تريد رمز `404`. الأداة جاهزة في فصل الاستثناءات. اللغة هنا: الرفض يسافر مع الـ Promise.

`try/catch` حول `await` يمسك الرفض في هذه الطبقة:

```ts
try {
  await requireBook('missing');
} catch (error) {
  // سجّل أو أعد رميًا أوضح
}
```

إن أمسكت الخطأ وابتلعته، الفلتر لن يراه. هذا متسق مع ما قاله توثيق دورة الطلب: الاستثناء الممسوك لا يصل إلى Exception Filter.

## أخطاء شائعة

- `await` داخل حلقة على عمليات مستقلة يمكن أن تعمل معًا. `Promise.all` يشغّل مجموعة ثم تنتظرها مرة واحدة. لا تستخدمه على عمليات يجب أن تتسلسل بسبب أثر على البيانات نفسها من غير فهم الترتيب.
- اعتبار `async` يسرّع حسابًا ثقيلًا. الانتظار يحرر الخيط عند I/O فقط.
- إرجاع Promise من constructor. البناء يبقى متزامنًا. التهيئة غير المتزامنة خطاف `OnModuleInit` لاحقًا.

## متى نستخدم async؟

على كل دالة خدمة أو متحكم تنتظر I/O: قاعدة بيانات، ملف، HTTP خارجي.

## متى لا نستخدمه؟

على دالة حساب نقية ترجع عددًا فورًا. `async` زائد يجعل النوع Promise بلا سبب.

## مقارنة

| | `.then` | `async/await` |
| --- | --- | --- |
| المعنى | نفسه: Promise | نفسه بأسلوب متتابع |
| الخطأ | `.catch` | `try/catch` حول `await` |
| في أمثلة Nest | أقل | الشكل الغالب |

## تمارين

### تمرين سهل

ما نوع إرجاع دالة `async` تكتب `return 1`؟

<details>
<summary>الحل</summary>

`Promise<number>`.

</details>

### تمرين متوسط

ماذا يحدث لخيط JavaScript أثناء `await` على استعلام قاعدة بيانات؟

<details>
<summary>الحل</summary>

الدالة الحالية تتوقف عند السطر، والخيط يعود لحلقة الأحداث ويستطيع معالجة عمل آخر حتى تصل النتيجة.

</details>

### تمرين متقدم

متحكم فيه `try/catch` حول استدعاء الخدمة، والـ catch فارغ ويعيد `null`. لماذا لن يحوّل Exception Filter هذا الفشل إلى `404`؟

<details>
<summary>الحل</summary>

لأن الاستثناء مُمسك داخل الدالة. الفلتر يرى غير الممسوك. إرجاع `null` قرار هذه الدالة، وقد يصبح رد `200` بجسم فارغ إن لم تغيّر الرمز. أعد الرمي، أو ارمِ استثناء HTTP واضحًا واترك الفلتر أو الطبقة الافتراضية تشكله.

</details>

## أسئلة مراجعة

- ماذا يحرر `await`، وماذا لا يسرّع؟
- كيف يصبح `throw` داخل `async` رفض Promise؟
- لماذا لا نفتح اتصال قاعدة بيانات داخل constructor؟

## الخلاصة

`Promise` نتيجة لاحقة، و `await` يقرأها داخل دالة واحدة من غير حبس خيط Node أثناء الإدخال والإخراج. Nest تنتظر Promise المتحكم. ما تمسكه بـ `catch` لا يصل إلى فلتر الاستثناءات.

## ماذا بعد؟

الإطار يلصق وصفًا على الأصناف بدوال خاصة. هذه Decorators.

## المصادر الرسمية

- TypeScript — Async functions (عبر JavaScript): https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
- MDN — Promise: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
- NestJS — Exception filters (الاستثناء الممسوك لا يصل): https://docs.nestjs.com/exception-filters
- Node.js — Event loop: https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
