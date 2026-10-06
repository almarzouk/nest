# Lazy-loading Modules

## الهدف من هذا الدرس

تحميل وحدة عند أول حاجة بـ `LazyModuleLoader` لتقصير إقلاع دالة serverless، ومعرفة ما الذي لا يُسجَّل إن تأخر: متحكمات، بوابات، resolvers، وmiddleware.

## المشكلة التي يحلها

الإقلاع الافتراضي يبني كل الوحدات قبل أول عمل. في دالة تُستدعى نادرًا، زمن البرد يدفع ثمن ميزات لا تخص هذا المسار. التحميل الكسول يؤجل `import()` وبناء الوحدة حتى يصل الطلب المناسب. في تطبيق طويل العمر داخل حاوية، الربح صغير لأن الإقلاع مرة واحدة.

## الفكرة ببساطة

احقن `LazyModuleLoader` من `@nestjs/core`. عند الحاجة:

```ts
const { LazyModule } = await import('./lazy.module.js');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

الناتج `ModuleRef` لتلك الوحدة. أول `load` يبني ويخزن، والتالي يعيد المخزون بسرعة. خطافات دورة الحياة لا تُستدعى على الوحدة الكسولة وخدماتها. لا تجعلها `global`. المتحكمات داخلها لا تُسجَّل كمسارات بعد أن بدأ الاستماع.

## التعريف التقني

الوحدة ملف Nest عادي:

```ts
import { Module } from '@nestjs/common';
import { LazyService } from './lazy.service.js';

@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

بعد `load`:

```ts
const { LazyService } = await import('./lazy.service.js');
const lazyService = moduleRef.get(LazyService);
```

`load` يقبل خيارًا ثانيًا `{ logger: false }` لإسكات سجل التحميل.

التوثيق يحذر: التقنية مختلفة عن lazy routes في Angular رغم الاسم. الوحدات الكسولة تشارك رسم التطبيق مع ما بُني عند الإقلاع ومع ما يُحمَّل لاحقًا.

ما لا يعمل كمسارات متأخرة:

- Controllers في HTTP. Fastify لا يضيف مسارات بعد الجاهزية، والتسجيل المتأخر لا يفتح URL.
- Gateways وموضوعات بعض النواقل (Kafka و gRPC و RabbitMQ) التي تشترك قبل الاتصال.
- Resolvers في GraphQL لأن المخطط يُبنى من metadata المحمّلة مسبقًا.
- `Middleware` عبر `configure` على `NestModule` عند الطلب.

يمكن الحصول على المحمّل من `app.get(LazyModuleLoader)` في `main.ts` أيضًا.

مع webpack اضبط `module` على `esnext` و `moduleResolution` على `node` حتى ينقسم الملف. مشروع Nest الافتراضي على `tsc` يستعمل `import()` الديناميكي كما في ESM، ومع امتداد `.js` في المسار المحلي.

## Mental Model

```text
الإقلاع: وحدات الشجرة الساخنة فقط
أول حدث: import() -> load -> get(Service) -> عمل
ثاني حدث: load من الكاش
المسار HTTP الجديد؟ لا يُضاف بهذه الآلية
```

الكسول خدمة تُستدعى من كود يعمل، لا باب URL يظهر فجأة.

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';
import { LazyModuleLoader } from '@nestjs/core';

@Injectable()
export class ReportJob {
  constructor(private readonly lazyModuleLoader: LazyModuleLoader) {}

  async run(): Promise<string> {
    const { ReportModule } = await import('./report.module.js');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportModule);
    const { ReportService } = await import('./report.service.js');
    const reports = moduleRef.get(ReportService);
    return reports.build();
  }
}
```

`ReportModule` فيها `ReportService` فقط، بلا متحكم.

## شرح المثال

`import()` يؤجل تقييم الملف حتى `run`. `load` يدمجه في الرسم مرة. `get` يجلب Singleton تلك الوحدة. دالة serverless التي لا تبني تقارير لا تدفع ثمن المحرك في البرد. دالة الحاوية التي تبني التقارير كل دقيقة تدفع الثمن عند أول تشغيل ثم تستعمل الكاش.

## ماذا يحدث داخليًا؟

1. الإقلاع لا يرى `ReportModule` إن لم تُستورد ساكنًا.
2. أول `load` يسجل المزودات ويبنيها ويربط ما تستطيع من الرسم الموجود.
3. خطاف `onModuleInit` على `ReportService` لا يُستدعى. افتح الموارد داخل الدالة أو في بناء عادي.
4. النداء التالي يعيد الوحدة المخزنة.
5. متحكم داخل الملف لا يضيف `GET` بعد `listen`.
6. وسم الوحدة `global` لا يجعلها عامة بأثر رجعي.

## مثال عملي

عامل طابور يشغّل `invoice` أو `export` حسب الحقل `kind`. حمّل `InvoiceModule` أو `ExportModule` داخل المعالج. أبقِ متحكم HTTP للمسار في الشجرة الساخنة. شارك `PrismaModule` عبر استيراده استيرادًا ساكنًا من الوحدة الكسولة إن احتاجت العميل، أو مرر البيانات إلى دالة لا تفتح اتصالًا جديدًا. لا تعتمد على `onModuleDestroy` في الوحدة الكسولة لإغلاق ما فتحته؛ أغلق في الدالة أو في خدمة ساخنة.

## أخطاء شائعة

- وضع `AppController` في وحدة كسولة وتوقع ظهور المسار.
- انتظار خطافات العمر.
- `load` لكل رسالة من غير الاستفادة من الكاش في التصميم (الكاش موجود، لكن `import` المتكرر ما زال يستحق الإبقاء على المرجع).
- اعتبار الميزة لازمة لتطبيق المتجر الدائم. الإقلاع مرة لا يبرر التعقيد.
- نسيان `.js` في مسار `import()` تحت ESM.

## متى نستخدمه؟

دالة أو عامل بارد يتفرع إلى خدمات ثقيلة حسب المدخل، والمسار الشبكي مسجّل مسبقًا.

## متى لا نستخدمه؟

لتقسيم متحكمات REST في عملية تستمع دائمًا. استيراد ساكن أوضح. ولا لمخطط GraphQL code-first.

## مقارنة

| التحميل | متى تُبنى الوحدة | المسارات |
| --- | --- | --- |
| ساكن في `imports` | أثناء الإقلاع | تُسجَّل قبل الاستماع |
| `LazyModuleLoader` | أول `load` ثم كاش | لا متحكمات متأخرة |
| Dynamic `register` | أثناء الإقلاع أيضًا، بخيار | مسجّلة مع الشجرة |

## تمارين

### تمرين سهل

ماذا يرجع `load` وكيف تجلب الخدمة؟

<details>
<summary>الحل</summary>

`ModuleRef`. تستدعي `get` على رمز الخدمة بعد `import` لصنفها.

</details>

### تمرين متوسط

لماذا لا يُستعمل هذا الأسلوب لإضافة `GET /reports` بعد الإقلاع؟

<details>
<summary>الحل</summary>

المتحكمات والـ middleware والبوابات وresolvers لا تُسجَّل كما يُتوقع عند التحميل المتأخر. منصات مثل Fastify لا تفتح مسارات بعد الجاهزية.

</details>

### تمرين متقدم

أين تغلق عميلًا فتحته خدمة داخل وحدة كسولة؟

<details>
<summary>الحل</summary>

في مسار الدالة التي استدعت الخدمة، أو في مزود ساخن يملك خطاف `onModuleDestroy`. خطافات الوحدة الكسولة لا تُستدعى.

</details>

## أسئلة مراجعة

- ما الفرق عن كسول Angular؟
- ماذا يحدث في ثاني `load`؟
- أي تطبيقات يبرر لها التوثيق هذا الأسلوب؟

## الخلاصة

`LazyModuleLoader` يبني وحدة خدمة عند الحاجة ويخزنها. الإقلاع البارد يستفيد. المسارات والمخطط والخطافات تبقى على الشجرة التي تُحمَّل قبل الاستماع.

## ماذا بعد؟

Asynchronous Providers: مصنع مزود يرجع وعدًا داخل الوحدة، وهو غير `forRootAsync`.

## المصادر الرسمية

- NestJS — Lazy loading modules: https://docs.nestjs.com/fundamentals/lazy-loading-modules
