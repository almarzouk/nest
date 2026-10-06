# Response Object و Library-specific Mode

## الهدف من هذا الدرس

معرفة متى يحقن `@Res()` كائن استجابة المنصة، وما الذي يتغير في من يكتب الاستجابة.

## المشكلة التي يحلها

تصدير CSV أو ضبط cookie يحتاج API المنصة: `res.send` أو `res.cookie`. استخدام ذلك في كل المسارات يعطل الأسلوب الذي يعيد قيمة ويترك Nest تكتب JSON. Nest تميّز الوضعين حتى لا يختلط «أنا أكتب الرد» مع «أنا أعيد قيمة».

## الفكرة ببساطة

`@Res()` أو `@Response()` يضع المعالج في وضع المكتبة. أنت تستدعي `res.send` أو `res.json`، وإلا يبقى الطلب معلقًا. `@Res({ passthrough: true })` يتيح لمس الكائن مع إبقاء Nest هي من ترسل القيمة المعادة.

من غير `passthrough` تفقد سلوك Interceptors والتسلسل القياسي على هذا المسار.

## التعريف التقني

وجود `@Res()` أو `@Next()` في التوقيع يعطل الأسلوب القياسي لذلك المعالج. التوثيق يحذّر من الجمع بينهما وبين `return` من غير `passthrough`: القيمة المعادة لا تُرسل كما تتوقع.

```ts
@Get('catalog.csv')
csv(@Res() res: Response) {
  res.type('text/csv');
  res.send('isbn,title\n');
}
```

```ts
@Post('ready')
ready(@Res({ passthrough: true }) res: Response) {
  res.cookie('report', 'ready');
  return { ok: true };
}
```

`@Res()` اسم مختصر لـ `@Response()`. النوع من المنصة: `import type { Response } from 'express'` مع `@types/express`. على Fastify استخدم نوع رد Fastify.

`passthrough: true` للإعداد الضيق: cookie أو رأس، ثم `return` حتى تكمل Nest الرمز والتسلسل والـ Interceptors التي تعتمد على القيمة.

`@Next()` نادر في Controller ويقلب الوضع نفسه. تمرير التحكم بأسلوب Express middleware داخل المعالج يخرج عن مسار هذا الفصل.

## Mental Model

```text
@Res()                         أنت تكتب النهاية
@Res({ passthrough: true })    أنت تضبط تفصيلًا، وNest تكمل
return value                   Nest تكتب، من غير حقن res
```

## أبسط مثال

```ts
import { Controller, Get, Res } from '@nestjs/common';
import type { Response } from 'express';

@Controller('reports')
export class CsvReportsController {
  @Get('catalog.csv')
  csv(@Res() res: Response) {
    res.type('text/csv');
    res.send('isbn,title\n9780001,أطلس\n');
  }
}
```

## شرح المثال

- لا `return` يُعتمد عليه. `res.send` هو نهاية الاستجابة.
- `res.type` يضبط `Content-Type` قبل الإرسال.
- نسيان `send` أو `json` أو `end` يترك الاتصال مفتوحًا.
- هذا المسار يناسب ملفًا خامًا، لا كائنًا تريد Nest أن تسلسله.

## ماذا يحدث داخليًا؟

Nest تكشف `@Res()` أو `@Next()` عند بناء المعالج وتوقف `RouterResponseController` عن إرسال القيمة المعادة. Interceptors التي تغلف القيمة المعادة أو تعتمد على التسلسل القياسي لا ترى المسار المعتاد، لأن الرد خرج من `res.send` خارج ذلك التعاقد.

مع `passthrough: true` يبقى الإرسال القياسي. يمكنك ضبط cookie أو رأس، ثم تعيد كائنًا فيصبح JSON، أو تعيد نصًا فيُرسل كما هو، ويبقى رمز POST الافتراضي 201 قابلًا للتعديل بـ `@HttpCode`.

كتابة `res.status(200).json(...)` من غير passthrough تعني أن الرمز والجسم قرارك، بما في ذلك POST الذي لم يعد يأخذ 201 تلقائيًا.

## مثال عملي

```ts
import { Controller, Get, Post, Res } from '@nestjs/common';
import type { Response } from 'express';
import { ReportsService } from './reports.service.js';

@Controller('reports')
export class ReportDownloadsController {
  constructor(private readonly reports: ReportsService) {}

  @Get('stock.csv')
  stock(@Res() res: Response) {
    const csv = this.reports.stockCsv();
    res.type('text/csv');
    res.send(csv);
  }

  @Post('ready')
  ready(@Res({ passthrough: true }) res: Response) {
    res.cookie('report', 'ready', { httpOnly: true });
    return { ok: true };
  }
}
```

`stock` وضع مكتبة: الخدمة تجهز النص، و `res.send` يكتبه. `ready` يضبط cookie ثم يعيد `{ ok: true }` فترسله Nest JSON مع رمز 201.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `@Res() res` مع `return { ok: true }` من غير passthrough | الكائن لا يُرسل بالأسلوب القياسي وقد يعلق الطلب إن لم يُستدعَ `send` |
| نسيان `res.send` في وضع المكتبة | العميل ينتظر إلى أن تنتهي المهلة |
| `@Res()` على مسار JSON عادي | تفقد التسلسل و Interceptors ذلك المسار |
| توقع 201 بعد `res.status(200).json` على POST | الرمز الذي كتبته على `res` هو الذي يخرج |

## متى نستخدمه؟

`@Res()` لرد خام تبنيه API المنصة: CSV أو بث تكتبه بنفسك. `passthrough: true` عندما تحتاج cookie أو رأسًا ثم تُكمل Nest إرسال القيمة.

## متى لا نستخدمه؟

لكائن أو مصفوفة JSON معتادة. أعد القيمة. لرمز ثابت أو تحويل، `@HttpCode` و `@Redirect` يبقيان في الأسلوب القياسي. رأس ثابت مثل `Cache-Control` يُكتب بـ `@Header` من غير حقن `res`.

## مقارنة

| التوقيع | من يكتب الجسم |
| --- | --- |
| `return value` | Nest، مع التسلسل و Interceptors |
| `@Res() res` ثم `res.send` | معالجك، والأسلوب القياسي متوقف |
| `@Res({ passthrough: true })` ثم `return` | أنت تضبط الكائن، وNest ترسل القيمة |

## تمارين

### تمرين سهل

ماذا يحدث لـ `GET` يستخدم `@Res()` ولا يستدعي طريقة إرسال؟

<details><summary>الحل</summary>

الطلب يبقى معلقًا. في وضع المكتبة إرسال الجسم مسؤوليتك.

</details>

### تمرين متوسط

تريد cookie `report=ready` وجسم `{ ok: true }` كـ JSON. أي شكل لـ `@Res`؟

<details><summary>الحل</summary>

`@Res({ passthrough: true })` ثم `res.cookie(...)` ثم `return { ok: true }`.

</details>

### تمرين متقدم

لماذا يفقد مسار `@Res()` من غير passthrough سلوك Interceptors القياسي؟

<details><summary>الحل</summary>

Nest تعطّل أسلوبها لهذا المعالج عندما ترى `@Res()` أو `@Next()`. الجسم يخرج عبر كائن المنصة، فسلسلة التسلسل والـ Interceptors التي تعتمد على القيمة المعادة لا تعمل على هذا المسار كما في `return`.

</details>

## أسئلة مراجعة

- ما مرادف `@Res()`؟
- ماذا يفعل `passthrough: true`؟
- أي Decorator آخر يضع المعالج في وضع المكتبة؟

## الخلاصة

`@Res()` يعني أنك تكتب الاستجابة بأدوات المنصة. `passthrough: true` يعيد الإرسال إلى Nest بعد لمسة على الكائن. مسارات JSON العادية تعيد قيمة وتترك الوضع القياسي.

## ماذا بعد؟

ماذا تفعل Nest عندما تعيد القيمة: Standard Nest Response Handling.

## المصادر الرسمية

- NestJS — Controllers (Library-specific): https://docs.nestjs.com/controllers#routing
- NestJS — Status code: https://docs.nestjs.com/controllers#status-code
