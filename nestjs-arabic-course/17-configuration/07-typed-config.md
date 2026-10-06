# Typed Config

## الهدف من هذا الدرس

جعل المترجم يرفض مفتاح إعداد غير موجود، ويستنتج نوع القيمة من واجهة تصف البيئة.

## المشكلة التي يحلها

`get("DATABSE_URL")` خطأ إملائي يُترجم بنجاح ويرجع `undefined` عند التشغيل. `get<number>("PORT")` الذي تكتبه بيدك قد يكذب: البيئة ما زالت نصًا إن لم يحوّلها المخطط. الواجهة مع `infer` تربط الاسم المسموح والنوع المعلن في مكان واحد.

## الفكرة ببساطة

عرّف واجهة بالمفاتيح التي تقرأها. مرّرها وسيطًا أول إلى `ConfigService<EnvironmentVariables>`. في `get` مرّر `{ infer: true }` ليستنتج النوع من الواجهة. وسيط ثانٍ `true` يزيل `undefined` من الإرجاع حين تكون القيم مضمونة بعد التحقق.

## التعريف التقني

من صفحة Configuration، للخدمة وسيطان نوعيان اختياريان:

```ts
constructor(
  private readonly config: ConfigService<EnvironmentVariables, true>,
) {}

const port = this.config.get("PORT", { infer: true });
```

الوسيط الأول يقيّد المفاتيح. `get("URL")` يفشل الترجمة إن لم توجد `URL` في الواجهة. مع `infer: true`، نوع `PORT` يصبح `number` إن أعلنتها `number` في الواجهة. مع `strictNullChecks` ومن غير الوسيط الثاني، النوع `number | undefined`. الوسيط الثاني `true` تأكيد بأن الطرق لا ترجع `undefined`.

النقطة تعمل مع كائن متداخل معلن في الواجهة:

```ts
ConfigService<{ database: { host: string } }>
this.config.get("database.host", { infer: true });
```

هذا تلميح ترجمة. التشغيل يبقى على ما خزّنه `forRoot` بعد المخطط. إن أعلنت `PORT: number` بينما Zod لم يُجبر القيمة، النوع يكذب وقت التشغيل. اجعل الواجهة مرآة لناتج `validationSchema`.

`ConfigType<typeof databaseConfig>` من الدرس السابق نوع لكائن نطاق واحد. `ConfigService<...>{}` نوع لخدمة القراءة العامة. استخدم الاثنين معًا: النطاق لخيارات القاعدة، والواجهة للمفاتيح المسطحة التي تُقرأ في `main.ts`.

## Mental Model

```text
واجهة EnvironmentVariables
  PORT: number
  DATABASE_URL: string
        |
ConfigService<EnvironmentVariables, true>
        |
get("PORT", { infer: true })  -> number
get("DATABSE_URL")            -> خطأ ترجمة
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";

interface LibraryEnv {
  PORT: number;
  DATABASE_URL: string;
  LIBRARY_NAME: string;
}

@Injectable()
export class LibraryRuntime {
  constructor(private readonly config: ConfigService<LibraryEnv, true>) {}

  listenPort(): number {
    return this.config.get("PORT", { infer: true });
  }

  databaseUrl(): string {
    return this.config.get("DATABASE_URL", { infer: true });
  }
}
```

## شرح المثال

`LibraryEnv` تذكر ثلاثة مفاتيح فقط. أي اسم آخر على `get` يرفضه المترجم مع `infer`. الوسيط `true` يناسب تطبيقًا يمرّر `validationSchema` يجعل هذه المفاتيح موجودة بعد الإقلاع. من غير المخطط، `true` وعد لا يفرضه التشغيل.

`LIBRARY_NAME` إن كان له `default` في Zod فهو نص مضمون. اربطه في الواجهة بالنوع الذي يرجع به المخطط.

## ماذا يحدث داخليًا؟

TypeScript يمحو الواجهة. لا يُضاف فحص مفاتيح عند التشغيل بسبب `ConfigService<LibraryEnv>`. الحماية التشغيلية تبقى `validationSchema` أو `getOrThrow`. `infer` يغيّر النوع المعروض في المحرر فقط، بقراءة حقل الواجهة المقابل للاسم الحرفي. اسم مبني في متغير `const key: string` يفقد الضيق، فتجنب المفاتيح الديناميكية على هذه الخدمة.

`ConfigType` يشتق حقول المصنع من نوع رجوع `registerAs`، لذلك تغيير المصنع يحدّث المستهلكين عند الترجمة.

## مثال عملي

بعد مخطط Zod الذي يجبر `PORT` ويطلب `DATABASE_URL`، انسخ شكل الناتج إلى `LibraryEnv`. `LibraryRuntime` يقرأ المنفذ لـ `app.listen` وعنوان القاعدة لدرس Prisma التالي. محاولة `get("DB_URL", { infer: true })` تفشل قبل التشغيل لأن الاسم ليس في الواجهة. هذا أرخص من اكتشاف الخطأ على الخادم.

لا تكرر الواجهة يدويًا إن استطعت اشتقاقها من `z.infer<typeof envSchema>`. إن بقي المخطط مصدر الحقيقة، الواجهة لا تُكتب مرتين.

## أخطاء شائعة

- واجهة تقول `PORT: number` ومخطط يترك `z.string()`.
- `true` كوسيط ثانٍ من غير تحقق إقلاع، ثم قراءة مفتاح غائب.
- تمرير مفتاح في متغير واسع فيتعطل `infer`.
- خلط `ConfigType` لكائن النطاق مع واجهة مسطحة ثم طلب `database.host` من واجهة لا تعلن `database`.

## متى نستخدمه؟

حين كثرت المفاتيح وظهر خطأ إملائي في مراجعة أو في التشغيل. الواجهة رخيصة بعد أن يوجد مخطط.

## متى لا نستخدمه؟

قبل وجود قائمة مفاتيح ثابتة. خدمة بثلاثة `get` واضحة لا تحتاج وسيطين نوعيين في أول يوم.

## مقارنة

| `get<string>("PORT")` | `get("PORT", { infer: true })` |
| --- | --- |
| أنت تختار النوع | الواجهة تختار النوع |
| المفتاح أي نص | المفتاح حرفي معروف |
| لا علاقة له بالمخطط | صادق إن طابق المخطط |

## تمارين

### تمرين سهل

ماذا يفعل `{ infer: true }` إن كانت `TIMEOUT` في الواجهة من نوع `string`؟

<details>
<summary>الحل</summary>

يجعل نوع القيمة المرجعة `string`، أو `string | undefined` إن لم تمرّر `true` وسيطًا ثانيًا للخدمة ومع `strictNullChecks`.

</details>

### تمرين متوسط

لماذا لا يكفي الوسيط النوعي ليوقف الإقلاع عند غياب `DATABASE_URL`؟

<details>
<summary>الحل</summary>

الواجهة تُمحى بعد الترجمة. الإيقاف يتم في `validationSchema` أو `getOrThrow` أثناء التشغيل.

</details>

### تمرين متقدم

كيف تتجنب كتابة `LibraryEnv` و `envSchema` بشكلين مختلفين؟

<details>
<summary>الحل</summary>

اجعل `z.infer<typeof envSchema>` نوع البيئة، واستخدمه وسيطًا لـ `ConfigService`. المخطط يبقى مصدر التحويل والوجود، والنوع يبقى ظلًا له.

</details>

## أسئلة مراجعة

- ما دور الوسيط النوعي الثاني `true`؟
- لماذا يكذب `PORT: number` إن لم يوجد `coerce`؟
- أين يعيش النوع المشتق من `registerAs`؟

## الخلاصة

Typed Config عقد ترجمة على أسماء المفاتيح وأنواعها. اربطه بمخطط التحقق حتى يطابق التشغيل ما يراه المترجم.

## ماذا بعد؟

[forRootAsync: أول استخدام حقيقي](./08-for-root-async-preview.md) يمرّر هذه القيم إلى وحدة تُبنى خياراتها أثناء الإقلاع.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
