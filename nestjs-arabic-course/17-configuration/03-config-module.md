# ConfigModule

## الهدف من هذا الدرس

تسجيل تحميل البيئة مرة واحدة في جذر التطبيق عبر `ConfigModule.forRoot`، ومعرفة متى تكون الوحدة عامة.

## المشكلة التي يحلها

استدعاء `dotenv.config()` داخل كل وحدة ميزة يحمّل الملف مرات، ويخفي ترتيب الدمج. وحدة لا تستورد مصدر الإعداد ثم تحقن `ConfigService` تفشل عند الإقلاع لأن المزوّد غير مرئي في نطاقها.

## الفكرة ببساطة

`ConfigModule.forRoot()` وحدة ديناميكية: تقرأ `.env`، تدمج البيئة، وتسجّل `ConfigService`. تضعها في `AppModule`. `isGlobal: true` يجعل المزوّد مرئيًا لبقية الوحدات بعد هذا التسجيل الواحد. من غير الخيار، كل وحدة ميزة تستورد `ConfigModule` قبل أن تحقن الخدمة.

## التعريف التقني

الحزمة `@nestjs/config`. الشكل المعتاد:

```ts
ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: ".env",
  cache: true,
});
```

`cache: true` يخزّن قراءات `process.env` خلف `get` لأن الوصول المتكرر إلى البيئة قد يكون مكلفًا. `load` يقبل مصفوفة دوال تبني كائنات إعداد متداخلة، ودرس namespace يفصّل `registerAs`.

`forFeature` تسجيل جزئي لملف إعداد خاص بوحدة ميزة، بعد أن يكون `forRoot` قد حمّل البيئة في الجذر. التوثيق يحذّر أن قيم `forFeature` قد لا تكون جاهزة في بنّاء وحدة أخرى؛ `onModuleInit` يعمل بعد تهيئة الوحدات التي تعتمد عليها.

`ConditionalModule.registerWhen(FooModule, "USE_FOO")` يحمّل وحدة إن لم يكن المتغير `false`. يحتاج `ConfigModule` محمّلًا حتى يُحسم `ConfigModule.envVariablesLoaded`. هذا أداة لاحقة، لا بديل عن التحقق من المفاتيح المطلوبة.

## Mental Model

```text
AppModule
  imports ConfigModule.forRoot({ isGlobal: true })
       |
       +-- يسجل ConfigService مرة
       |
BooksModule, LoansModule
  تحقن ConfigService من غير import إضافي
```

## أبسط مثال

```ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { BooksModule } from "./books/books.module.js";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      cache: true,
    }),
    BooksModule,
  ],
})
export class AppModule {}
```

## شرح المثال

`forRoot` يُستدعى في الجذر فقط. `BooksModule` لا تعيد الاستدعاء ولا تستورد `ConfigModule` لأن `isGlobal` صدّر الخدمة للتطبيق بعد تسجيلها في `AppModule`. `cache` يخص قراءات البيئة المتكررة، ولا يغني عن التحقق من الشكل.

ترتيب `imports` هنا يضع الإعداد قبل وحدات الميزة. القيم المطلوبة ما زالت تُفحص في درس التحقق، لأن غياب الملف وحده لا يوقف الإقلاع.

## ماذا يحدث داخليًا؟

`forRoot` ترجع Dynamic Module فيها مزوّد `ConfigService`. أثناء التهيئة تُحلَّل ملفات البيئة وتُدمج. إن وُجد `validationSchema` أو `validate`، الفشل يرمي قبل أن يكتمل الإقلاع. درس لاحق يكتب ذلك المخطط.

`isGlobal` لا يعني أن كل الوحدات في المشروع يجب أن تكون عامة. هي استثناء لمزوّد يحتاجه الجميع بعد تكوين واحد. بقية ميزات المكتبة، مثل الكتب والإعارات، تبقى وحدات عادية تُستورد حيث تُستخدم.

## مثال عملي

مكتبة الحي: `AppModule` يستورد `ConfigModule.forRoot({ isGlobal: true })` ثم `BooksModule` و `LoansModule`. خدمة الإعارة تحقن `ConfigService` لتقرأ `LOAN_DAYS` إن كان مدة الإعارة تختلف بين فرع تجريبي وفرع حقيقي. إن أزلت `isGlobal` ولم تستورد `ConfigModule` داخل `LoansModule`، الإقلاع يفشل لأن `ConfigService` غير متاح في ذلك النطاق.

في `main.ts`، بعد `NestFactory.create`، تستطيع `app.get(ConfigService)` لقراءة `PORT` قبل `listen`. ذلك وصول من حاوية التطبيق، لا استيراد للخدمة من ملف الإقلاع كأنها دالة حرة.

## أخطاء شائعة

- استدعاء `forRoot` داخل كل Feature Module فيتكرر التحميل.
- `isGlobal: true` من غير تسجيل الوحدة في `AppModule`.
- جعل كل وحدات المجال `@Global()` لأن الإعداد صار عامًا.
- قراءة `process.env` في جسم الوحدة عند الاستيراد ثم افتراض أن `forRoot` سبقه. تقييم الملف يحدث عند الاستيراد، والتحميل داخل التهيئة.

## متى نستخدمه؟

مرة في الجذر لكل تطبيق يقرأ بيئة. `isGlobal: true` حين معظم الوحدات تحتاج `ConfigService`.

## متى لا نستخدمه؟

لإخفاء أسرار داخل مزوّدات بلا تحقق. التسجيل العام يسهّل الحقن، ولا يثبت أن المفتاح موجود أو أن نوعه صحيح.

## مقارنة

| `forRoot` في الجذر | استيراد `ConfigModule` في كل ميزة |
| --- | --- |
| تكوين واحد مع `isGlobal` | صريح حين وحدات قليلة تحتاج الإعداد |
| المزوّد مرئي بعد التسجيل | كل وحدة تذكر الاعتماد |

## تمارين

### تمرين سهل

أين يُستدعى `ConfigModule.forRoot` في تطبيق المكتبة؟

<details>
<summary>الحل</summary>

في `AppModule` مرة واحدة. وحدات الكتب والإعارة تحقن `ConfigService` ولا تعيد `forRoot`.

</details>

### تمرين متوسط

`isGlobal: true` مضبوط، لكن `BooksService` لا يرى `ConfigService`. ما أول شيء تفحصه؟

<details>
<summary>الحل</summary>

هل `AppModule` يستورد ناتج `forRoot` فعلًا؟ العام يبدأ بعد تسجيل الوحدة في الجذر. ثم تتأكد أن الحقن في بنّاء مزوّد مسجّل، لا في دالة خارج الحاوية.

</details>

### تمرين متقدم

لماذا قراءة `process.env.DATABASE_URL` في أعلى `app.module.ts` لتمريرها إلى وحدة أخرى قد ترى سلسلة فارغة رغم وجود `.env`؟

<details>
<summary>الحل</summary>

أعلى الملف يُقيَّم عند الاستيراد، وقد يسبق تحميل dotenv داخل `forRoot`. مرّر القراءة إلى مصنع يعمل أثناء التهيئة، وهو موضوع `forRootAsync` في آخر الفصل. أو حمّل الملف قبل الإقلاع بعلم `--env-file`.

</details>

## أسئلة مراجعة

- ماذا يسجّل `forRoot`؟
- متى تختار `isGlobal`؟
- لماذا لا يُستدعى `forRoot` في كل ميزة؟

## الخلاصة

`ConfigModule.forRoot` هو باب الإعداد في الجذر. `isGlobal` يوزّع `ConfigService` بعد تسجيل واحد. القراءة نفسها ما زالت تحتاج خدمة تعرف المفتاح والنوع.

## ماذا بعد؟

[ConfigService](./04-config-service.md) يقرأ المفتاح، والقيمة الافتراضية، و `getOrThrow`.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
