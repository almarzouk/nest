# Namespaced Config

## الهدف من هذا الدرس

تجميع إعداد ميزة واحدة تحت اسم، مثل `database`، وحقنه ككائن بدل قراءة مفاتيح مبعثرة.

## المشكلة التي يحلها

`DATABASE_HOST` و `DATABASE_PORT` و `DATABASE_NAME` تتكرر في خدمات الكتب والإعارات والتقارير. كل خدمة تعيد `parseInt` على المنفذ. حين تنتقل القاعدة إلى عنوان واحد `DATABASE_URL`، البحث يلمس كل الملفات.

## الفكرة ببساطة

`registerAs("database", factory)` يبني كائنًا ويسمّيه. `load: [databaseConfig]` يدمجه عند `forRoot`. القراءة تكون `config.get("database.host")` أو بحقن الرمز `databaseConfig.KEY` مع النوع `ConfigType`.

## التعريف التقني

`registerAs` مُصدَّر من `@nestjs/config`. داخل المصنع، `process.env` يحتوي المفاتيح بعد دمج `.env` وبيئة التشغيل. المصنع مكان التحويل: `parseInt` للمنفذ، ورفض القيمة غير العددية.

```ts
export default registerAs("database", () => ({
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT ?? "", 10) || 5432,
}));
```

`ConfigModule.forFeature(databaseConfig)` تسجيل جزئي داخل وحدة الميزة إن كان الملف يعيش معها، بعد `forRoot` في الجذر.

`databaseConfig.asProvider()` يحوّل النطاق إلى تعريف مزوّد جاهز لـ `forRootAsync` في وحدة أخرى:

```ts
{
  imports: [ConfigModule.forFeature(databaseConfig)],
  useFactory: (configuration) => configuration,
  inject: [databaseConfig.KEY],
}
```

هكذا تصل خيارات القاعدة إلى `TypeOrmModule.forRootAsync` من غير نسخ المصنع. درس `forRootAsync` يفتح هذا الشكل. هنا نكتفي ببناء النطاق وقراءته.

`validationSchema` لا يفحص الكائن الناتج. افحص الحقول داخل المصنع أو افحص المتغيرات الخام في المخطط ثم ابنِ الكائن منها.

## Mental Model

```text
DATABASE_HOST, DATABASE_PORT
        |
 registerAs("database")
        |
 { host, port }
        |
 get("database.host")    أو    @Inject(databaseConfig.KEY)
```

## أبسط مثال

```ts
import { registerAs } from "@nestjs/config";

export const databaseConfig = registerAs("database", () => {
  const port = Number(process.env.DATABASE_PORT ?? "5432");
  if (!Number.isInteger(port)) {
    throw new Error("DATABASE_PORT must be an integer");
  }
  return {
    host: process.env.DATABASE_HOST ?? "localhost",
    port,
    name: process.env.DATABASE_NAME ?? "library",
  };
});
```

```ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { databaseConfig } from "./database.config.js";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

## شرح المثال

الاسم `"database"` هو بادئة القراءة المنقّطة. `host` الافتراضي `localhost` مقبول للتطوير، واسم القاعدة `library` كذلك. لا تضع كلمة المرور في افتراضي. إن كان الإنتاج يعتمد على `DATABASE_URL` واحد، المصنع يقرأ ذلك المفتاح ويرجع `{ url }` بدل ثلاثة حقول.

رمي الخطأ من المصنع يوقف الإقلاع لأن `load` يعمل أثناء تهيئة `ConfigModule`.

## ماذا يحدث داخليًا؟

كل دالة في `load` تُستدعى وتُخزَّن نتيجتها تحت اسمها إن كانت ناتج `registerAs`، أو تُدمج كجذر إن كانت مصنعًا بلا اسم. `get("database.port")` ينزل في الكائن. حقن `databaseConfig.KEY` يعيد الكائن كاملًا بنوع `ConfigType<typeof databaseConfig>`، فيرى المترجم `port` و `host` ولا يقبل مفتاحًا مخترعًا على هذا الكائن.

`forFeature` يحمّل النطاق أثناء تهيئة تلك الوحدة. التوثيق يطلب قراءة هذه القيم من `onModuleInit` لا من بنّاء وحدة أخرى قد تُهيّأ قبلها.

## مثال عملي

`catalog.config.ts` يجمع `CATALOG_URL` و `CATALOG_TIMEOUT_MS` تحت اسم `catalog`. خدمة الفهرس تحقن `ConfigType` مرة واحدة وتبني `HttpModule` أو عميلًا بمهلة عددية. خدمة الكتب لا تعرف أسماء المتغيرات. حين تنتقل المهلة من `3000` إلى قيمة بيئة، يتغيّر ملف النطاق فقط.

## أخطاء شائعة

- نسيان `load: [databaseConfig]` ثم قراءة `database.host` والحصول على `undefined`.
- تكرار `parseInt` في الخدمات بعد أن أصبح مكانه المصنع.
- افتراض أن مخطط Zod على البيئة يحوّل حقول الكائن المسمّى تلقائيًا.
- استدعاء `forFeature` من غير `forRoot` في الجذر، فيغيب دمج `.env` قبل المصنع.

## متى نستخدمه؟

عندما لميزة أكثر من مفتاح مترابط: اتصال قاعدة، عميل HTTP، بريد.

## متى لا نستخدمه؟

لمفتاح واحد مثل `PORT` يُقرأ في `main.ts`. نطاق من حقل واحد يزيد ملفاتًا بلا تجميع حقيقي.

## مقارنة

| مفاتيح مسطحة | `registerAs` |
| --- | --- |
| `get("DATABASE_PORT")` في كل خدمة | كائن `database` يُحقن مرة |
| التحويل مبعثر | التحويل في المصنع |
| مناسب لمفتاح وحيد | مناسب لخيارات ميزة |

## تمارين

### تمرين سهل

ما وسيط `registerAs` الأول، وماذا تطلب من `ConfigService` لقراءة `port`؟

<details>
<summary>الحل</summary>

الأول اسم النطاق، مثل `"database"`. القراءة `get("database.port")` أو حقن `databaseConfig.KEY`.

</details>

### تمرين متوسط

`asProvider()` يرجع `imports` و `useFactory` و `inject`. لماذا يوجد `forFeature` داخل `imports`؟

<details>
<summary>الحل</summary>

حتى تُحمَّل دالة النطاق في الوحدة التي ستستهلك الكائن، ويُحقن `databaseConfig.KEY` في المصنع. هذا شكل `forRootAsync` الذي يستلم خيارات جاهزة.

</details>

### تمرين متقدم

وحدتان تستدعيان `ConfigModule.forFeature(databaseConfig)`، ووحدة ثالثة تقرأ الكائن من بنّائها فترى `undefined`. ما توجيه التوثيق؟

<details>
<summary>الحل</summary>

ترتيب تهيئة الوحدات غير مضمون. اقرأ ناتج التسجيل الجزئي في `onModuleInit` بعد تهيئة الاعتمادات، أو حمّل النطاق مرة عبر `load` في `forRoot` بالجذر إن كان كل التطبيق يحتاجه.

</details>

## أسئلة مراجعة

- أين يحدث `parseInt` لمنفذ القاعدة؟
- ما الفرق بين `get("DATABASE_PORT")` و `get("database.port")`؟
- هل `validationSchema` يفحص كائن `registerAs`؟

## الخلاصة

النطاق يجمع مفاتيح الميزة في مصنع واحد. القراءة المنقّطة أو `ConfigType` تمنع تكرار التحويل. تمرير هذا الكائن إلى وحدة أخرى هو دور `forRootAsync`.

## ماذا بعد؟

[Typed Config](./07-typed-config.md) يقيّد أسماء المفاتيح على `ConfigService` نفسه عبر واجهة و `infer`.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
