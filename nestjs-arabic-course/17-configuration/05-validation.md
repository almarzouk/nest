# التحقق من الإعدادات

## الهدف من هذا الدرس

رفض إقلاع المكتبة إذا غاب مفتاح مطلوب أو جاء `PORT` بما لا يُحوَّل إلى منفذ صالح.

## المشكلة التي يحلها

`get` يصمت عن الغياب حتى أول طلب يستخدم المفتاح. التطبيق يسمع على منفذ ثم يسقط حين تُفتح أول إعارة لأن `DATABASE_URL` فارغ. أسوأ من ذلك أن `"abc"` يمر كنص ويصبح `NaN` بعد `Number`.

التحقق عند `forRoot` يقلب الفشل إلى وقت الإقلاع، ويجمع كل الحقول الفاشلة في رسالة واحدة.

## الفكرة ببساطة

تمرّر مخططًا يصف شكل البيئة. الحزمة تمرّر المتغيرات إلى المخطط قبل أن يكتمل الإقلاع. إن رمى المخطط، `NestFactory.create` لا يصل إلى `listen`. القيم التي يرجعها المخطط، بعد التحويل، هي ما يقدّمه `ConfigService`.

## التعريف التقني

`validationSchema` يقبل مخطط Standard Schema: Zod و Valibot و ArkType، و Joi بدءًا من الإصدار 18. التوثيق يفضّل مكتبة حديثة للمشاريع الجديدة. مثال Zod:

```ts
ConfigModule.forRoot({
  validationSchema: z.object({
    NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
    PORT: z.coerce.number().default(3000),
  }),
});
```

متغيرات البيئة تصل نصوصًا، لذلك `z.coerce.number()` يحوّل `PORT`. مفتاح بلا `default` مطلوب، وغيابه يرمي أثناء الإقلاع. المفاتيح التي لا يعلنها المخطط تبقى متاحة عبر `ConfigService`. المفاتيح المجهولة مثل `PATH` لا تفشل التحقق افتراضيًا. كل حقل فاشل يُذكر، بصيغة `PATH: message`، وتُجمع الأسطر.

`validationOptions.libraryOptions` يمرّر خيارات المكتبة. لـ Joi تبقى الافتراضات التاريخية `allowUnknown: true` و `abortEarly: false` ما لم تغطّها.

البديل دالة `validate` متزامنة: تستقبل `Record<string, unknown>` وترجع الكائن بعد التحويل. إن رمت، لا يُقلع التطبيق. صفحة الإعداد تعرض ذلك مع `plainToInstance` و `validateSync` من class-validator. لا تستخدم الدالة `async`.

`validatePredefined: false` يعطّل فحص متغيرات كانت في `process.env` قبل استيراد الوحدة.

المخطط يفحص متغيرات البيئة، لا كائن `load` المخصص. فحص ملف YAML يتم داخل مصنعه.

## Mental Model

```text
.env + process.env
        |
   validationSchema
        | فشل -> استثناء، لا listen
        | نجاح -> قيم محوّلة
   ConfigService.get("PORT")  // عدد إن أجبره Zod
```

## أبسط مثال

```ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.string().min(1),
});

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: envSchema,
    }),
  ],
})
export class AppModule {}
```

## شرح المثال

`DATABASE_URL` بلا افتراضي: سلسلة فارغة أو غياب يوقف الإقلاع. `PORT` يُجبر إلى عدد ويُرفض خارج 1–65535، وغيابه يعني `3000` في التطوير. `NODE_ENV` مجموعة مغلقة حتى لا يمر `prod` بالخطأ ويختار مسار التطوير.

لا تضع السر الحقيقي في المخطط. المخطط يصف الشكل فقط.

## ماذا يحدث داخليًا؟

أثناء `forRoot` تُجمع البيئة ثم تُمرَّر إلى `~standard.validate` الخاص بالمخطط. الناتج المحوّل يحل محل القيم التي يخدمها `ConfigService` للمفاتيح المعلنة. لذلك `get("PORT")` بعد `z.coerce.number()` يمكن أن يكون عددًا، لا النص `"3000"`. هذا أثر المخطط، لا أثر الوسيط `get<number>` وحده.

رسالة الفشل تسمي الحقول. اقرأها قبل أن تضيف `default` لإسكاتها. الافتراضي على سر يخفي إنتاجًا بلا سر.

## مثال عملي

بيئة الاختبار تضبط `NODE_ENV=test` و `DATABASE_URL` لقاعدة `library_test`. إن نسي ملف CI المتغير الثاني، المهمة تفشل في ثوانٍ عند الإقلاع، لا بعد أول اختبار يكتب كتابًا. `PORT` في الاختبار قد يبقى على الافتراضي لأن الاختبار يستدعي `app.listen(0)` بنفسه ولا يستخدم إعداد المنفذ.

دالة `validate` تناسب حين تريد رسالة عربية واحدة أو رفض شرط يعبر مفتاحين، مثل «`REDIS_URL` مطلوب حين `CACHE=redis`». ارمِ `Error` من الدالة. أبقِها متزامنة.

## أخطاء شائعة

- `z.string()` على `PORT` ثم مفاجأة أن القيمة بقيت نصًا.
- `default` على `DATABASE_URL` بعنوان تطوير فيصل إلى صورة الإنتاج.
- افتراض أن `validationSchema` يفحص كائن `configuration.ts` المتداخل. الفحص هناك داخل المصنع.
- `validate` غير متزامن. الخيار يتوقع دالة متزامنة.

## متى نستخدمه؟

لكل مفتاح يتوقف التطبيق إن غاب أو تشوّه: عنوان القاعدة، سر التوقيع، المنفذ، `NODE_ENV`.

## متى لا نستخدمه؟

لفحص جسم `POST /books`. ذلك `ValidationPipe` على DTO. مخطط البيئة يفحص ظرف الإقلاع فقط.

## مقارنة

| `validationSchema` | دالة `validate` |
| --- | --- |
| Zod أو Joi 18 أو غيرهما من Standard Schema | أي فحص متزامن تكتبه |
| تحويلات المكتبة تصل إلى `ConfigService` | ترجع أنت الكائن النهائي |
| خيارات المكتبة تحت `libraryOptions` | الرسالة التي ترميها |

## تمارين

### تمرين سهل

لماذا `PORT` في مخطط Zod يستخدم `z.coerce.number()`؟

<details>
<summary>الحل</summary>

لأن البيئة تقدّم نصوصًا. `coerce` يحوّل `"3000"` إلى `3000` قبل فحص `int` والمدى.

</details>

### تمرين متوسط

التطبيق يقلع و `ConfigService.get("DATABASE_URL")` يرجع `""`. أين ثغرة المخطط؟

<details>
<summary>الحل</summary>

المفتاح إمّا غير موجود في `z.object`، أو مُعلن كـ `z.string()` بلا `min(1)`. السلسلة الفارغة ليست `undefined`، و `getOrThrow` لا يرفضها. أضف `min(1)` أو `.url()` بما يناسب شكل عنوانك.

</details>

### تمرين متقدم

تريد رفض الإقلاع إذا كان `NODE_ENV=production` و `DATABASE_URL` يحتوي `localhost`. أي أداة تختار، المخطط أم `validate`؟

<details>
<summary>الحل</summary>

الشرط يعبر مفتاحين. دالة `validate` المتزامنة تقرأ الكائن وترمّي `Error` واضحة. يمكن التعبير عنه في Zod بـ `superRefine` أيضًا إن أبقيت كل شيء في المخطط. المهم أن الفشل يحدث داخل `forRoot` قبل `listen`.

</details>

## أسئلة مراجعة

- متى يُرفض الإقلاع؟
- هل الوسم النوعي على `get` يغني عن المخطط؟
- ماذا يفحص `validationSchema`، البيئة أم ملف YAML؟

## الخلاصة

المخطط جزء من `forRoot`. النقص والتحويل الخطأ يظهران قبل أول طلب، و `ConfigService` يقدّم القيم بعد إكراه النوع.

## ماذا بعد؟

[Namespaced Config](./06-namespaced-config.md) يجمع مفاتيح القاعدة في كائن واحد باسم `database`.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
