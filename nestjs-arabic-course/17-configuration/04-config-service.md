# ConfigService

## الهدف من هذا الدرس

قراءة الإعداد المحمّل عبر `ConfigService.get` و `getOrThrow`، مع قيمة افتراضية آمنة حيث يناسب الافتراض.

## المشكلة التي يحلها

بعد `forRoot` تبقى القيم خلف الخدمة. استدعاء `process.env` من جديد يتجاوز الدمج والتحقق اللاحق، ويعيد النصوص الخام. المفتاح الناقص الذي يُقرأ بـ `get` يرجع `undefined`، وقد يصل `NaN` إلى `listen` إذا حوّلته من غير فحص.

## الفكرة ببساطة

تحقن `ConfigService` وتطلب مفتاحًا بالاسم. `get` يرجع القيمة أو الافتراضي الذي تمرّره. `getOrThrow` يرمي إن كانت النتيجة `undefined`، ونوع الإرجاع يستبعد `undefined`. الوسيط النوعي `get<string>` تلميح TypeScript، لا فحص تشغيل.

## التعريف التقني

استورد `ConfigService` من `@nestjs/config`. إن لم يكن `ConfigModule` عامًا، استورده في وحدة الميزة.

```ts
constructor(private readonly config: ConfigService) {}
```

`get` يقبل مفتاحًا، وافتراضيًا اختياريًا. النقطة في الاسم تقرأ كائنًا متداخلًا بُني عبر `load`، مثل `database.host`. `getOrThrow` بنفس الوسائط ويرمي إن بقيت القيمة `undefined`.

`skipProcessEnv: true` في `forRoot` يجعل `get` يتجاهل `process.env` الخام ويقرأ ملفات الإعداد المخصصة وما نتج عن التحقق. مفيد حين تريد مصدرًا واحدًا بعد التحويل.

في `main.ts`:

```ts
const config = app.get(ConfigService);
const port = config.get<string>("PORT");
```

## Mental Model

```text
ConfigService
  get("LIBRARY_NAME", "Library")  -> نص، وقد يكون الافتراضي
  getOrThrow("DATABASE_URL")      -> نص أو استثناء عند الطلب
  get("database.port")            -> حقل داخل كائن load
```

الافتراضي يناسب اسم المكتبة. عنوان القاعدة لا يناسبه افتراض صامت.

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";

@Injectable()
export class ListenPort {
  constructor(private readonly config: ConfigService) {}

  get port(): number {
    const raw = this.config.get<string>("PORT", "3000");
    const port = Number(raw);
    if (!Number.isInteger(port) || port < 1 || port > 65535) {
      throw new Error("PORT must be an integer from 1 to 65535");
    }
    return port;
  }
}
```

## شرح المثال

`PORT` إن غاب يصبح `"3000"` ثم `3000`. القيمة `"0"` أو `"abc"` تفشل الفحص هنا. هذا الفحص ما زال يدويًا؛ درس التحقق ينقله إلى مخطط يرفض الإقلاع قبل أول طلب.

`get<string>` لا يحوّل `"3000"` إلى عدد. النوع الذي تكتبه تلميح للمترجم. التحويل يبقى `Number` أو `z.coerce.number()` في المخطط.

## ماذا يحدث داخليًا؟

الخدمة تبحث في البنية التي بناها `forRoot`: ناتج الدمج، ثم مفاتيح الكائنات المحمّلة بالاسم المنقّط. مع `cache: true` تُحفظ قراءات مفاتيح `process.env`. `getOrThrow` يفحص `undefined` بعد الحل، بما في ذلك بعد الافتراضي إن استخدمته. سلسلة فارغة `""` ليست `undefined`، فلا يرمي `getOrThrow` بسببها. إن كان الفارغ غير مقبول، المخطط أو فحصك يرفضه.

حقن الخدمة في اختبار يتم باستبدال المزوّد:

```ts
{ provide: ConfigService, useValue: { get: () => "3000", getOrThrow: () => "3000" } }
```

الاختبار لا يلمس `.env` الحقيقي.

## مثال عملي

`BooksService` لا تقرأ البيئة. `CatalogClient` الذي ينادي فهرسًا خارجيًا يحقن `ConfigService` ويأخذ `CATALOG_URL` بـ `getOrThrow` لأن الغياب يعني إقلاعًا لا يعرف أين الفهرس. اسم العرض `LIBRARY_NAME` يُقرأ بـ `get("LIBRARY_NAME", "Neighborhood Library")` لأن الغياب له بديل مفهوم.

لا تطبع `CATALOG_URL` إن حمل رمزًا. سجّل «catalog client configured» من غير القيمة.

## أخطاء شائعة

- افتراض أن `get<number>("PORT")` يحوّل النص وقت التشغيل.
- `getOrThrow` على مفتاح فارغ السلسلة واعتبار ذلك حماية كافية.
- تمرير افتراضي لسر، فيعمل التطبيق بسر التطوير على الإنتاج.
- بناء كائن خيارات في عشر خدمات كل منها يستدعي `get` للمفتاح نفسه بشكل مختلف.

## متى نستخدمه؟

في مزوّد واحد يبني خيارات ميزة: منفذ الاستماع، عنوان عميل HTTP، علم تشغيل.

## متى لا نستخدمه؟

في كل دالة مجال. الكيان `Book` يستقبل عنوانًا وسعرًا جاهزين. من يقرأ الإعداد هو حد التركيب.

## مقارنة

| `get` | `getOrThrow` |
| --- | --- |
| يرجع قيمة أو `undefined` أو الافتراضي | يرمي إن كانت النتيجة `undefined` |
| مناسب لاسم له بديل | مناسب لعنوان قاعدة أو سر |

## تمارين

### تمرين سهل

ما نوع القيمة التي يرجعها `get("PORT")` قبل أي مخطط تحويل؟

<details>
<summary>الحل</summary>

نص من البيئة، أو `undefined` إن غاب المفتاح ولم تمرّر افتراضيًا. التلميح `<number>` لا يغيّر ذلك وحده.

</details>

### تمرين متوسط

لماذا افتراضي `"dev-secret"` لـ `JWT_SECRET` أخطر من غياب المفتاح؟

<details>
<summary>الحل</summary>

الغياب يفشل عند `getOrThrow` أو عند المخطط. الافتراضي يجعل الإنتاج يوقّع بسر معروف نُسخ من التطوير، والتطبيق يقلع كأنه سليم.

</details>

### تمرين متقدم

خدمة تقرأ `config.get("database.host")` وترى `undefined` رغم أن `DATABASE_HOST` موجود في `.env`. ما النقص؟

<details>
<summary>الحل</summary>

الاسم المنقّط يقرأ كائنًا بنيته دالة `load` أو `registerAs`، لا متغير البيئة الخام. المتغير الخام يُطلب بالاسم `DATABASE_HOST` إلى أن يربطه مصنع الإعداد بحقل `database.host`.

</details>

## أسئلة مراجعة

- ماذا يغيّر الوسيط النوعي في `get<string>`؟
- متى تختار `getOrThrow`؟
- كيف تختبر مستهلكًا للخدمة من غير ملف `.env`؟

## الخلاصة

`ConfigService` نقطة القراءة. `get` مع افتراضي للقيم التي لها بديل آمن، و `getOrThrow` لما لا يعمل التطبيق من دونه. التحويل والرفض المبكر موضوع التحقق.

## ماذا بعد؟

[التحقق من الإعدادات](./05-validation.md) يوقف الإقلاع إذا نقص مفتاح أو جاء المنفذ نصًا غير عددي.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
