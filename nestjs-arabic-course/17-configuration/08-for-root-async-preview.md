# forRootAsync: أول استخدام حقيقي

## الهدف من هذا الدرس

بناء خيارات وحدة أخرى من `ConfigService` أثناء التهيئة، عبر `forRootAsync`، لا من قيم تُقرأ عند استيراد الملف.

## المشكلة التي يحلها

`TypeOrmModule.forRoot({ url: process.env.DATABASE_URL })` يُقيَّم عندما يُستورد `app.module.ts`. في تلك اللحظة قد لا يكون `ConfigModule` قد دمج `.env` بعد، فيصل `url` فارغًا وتُحفظ تلك اللقطة داخل خيارات الوحدة. إعادة القراءة لاحقًا لا تصلح الخيار الذي بُني.

`forRootAsync` يؤخر المصنع حتى مرحلة التهيئة، ويحقن `ConfigService` بعد تحميل البيئة والتحقق.

## الفكرة ببساطة

`forRoot` يستقبل كائنًا جاهزًا. `forRootAsync` يستقبل مصنعًا. المصنع يمكن أن يكون `async`، ويُدرج في `inject` ما يحتاجه. Nest ينشئ الاعتمادات أولًا ثم يستدعي المصنع ويسجّل الوحدة بالناتج.

أول استخدام حقيقي في هذه الدورة: تمرير اتصال القاعدة. الفصل التالي يختار Prisma مسارًا تطبيقيًا. الشكل نفسه ينطبق على أي وحدة توثّق `forRootAsync`، ومنها `TypeOrmModule` في التوثيق الرسمي.

## التعريف التقني

من صفحة TypeORM، المصنع يحقن `ConfigService`:

```ts
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: "postgres",
    url: config.getOrThrow<string>("DATABASE_URL"),
    autoLoadEntities: true,
    synchronize: false,
  }),
});
```

`imports: [ConfigModule]` لازم إن لم تكن الوحدة عامة ومرئية. مع `isGlobal: true` قد يكفي الحقن، والإبقاء على `imports` يوثّق الاعتماد.

إن كان الاتصال في نطاق `registerAs`، التوثيق يختصر المصنع:

```ts
TypeOrmModule.forRootAsync(databaseConfig.asProvider());
```

`asProvider()` يرجِع `imports` فيه `ConfigModule.forFeature`، و `inject` لـ `databaseConfig.KEY`، و `useFactory` يمرّر الكائن كما هو. كائن النطاق يجب أن يطابق ما تنتظره `forRoot`، أو لفّه بمصنع يعدّل الشكل.

`forRootAsync` على `ConfigModule` نفسه أقل شيوعًا. التحميل يتم بـ `forRoot`. غير المتزامن يظهر عندما تعتمد وحدة ثانية على القيم.

لا تضع `synchronize: true` في هذا المصنع. توثيق TypeORM يحذّر أنه قد يمسح بيانات الإنتاج. الترحيل هو طريق تغيير المخطط، ويبدأ من الفصل التالي.

## Mental Model

```text
استيراد الملف          forRoot({ url: process.env... })  لقطة مبكرة
تهيئة الوحدات         forRootAsync -> inject ConfigService
                                      -> useFactory -> خيارات
                                      -> اتصال القاعدة
```

## أبسط مثال

```ts
import { Module } from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { TypeOrmModule } from "@nestjs/typeorm";

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: "postgres" as const,
        url: config.getOrThrow<string>("DATABASE_URL"),
        autoLoadEntities: true,
        synchronize: false,
      }),
    }),
  ],
})
export class AppModule {}
```

## شرح المثال

المثال معاينة، لا قرار بأن TypeORM مسار الدورة. المسار التطبيقي Prisma، و `PrismaService` سيقرأ `DATABASE_URL` من `ConfigService` داخل بنّائه، بعد أن يكون التحقق قد مر. `forRootAsync` هنا يوضح التوقيت: المصنع يعمل أثناء التهيئة، و `getOrThrow` يمنع اتصالًا بعنوان فارغ.

`synchronize: false` متعمَّد حتى لا يُنسخ تحذير التوثيق إلى مشروعك كإعداد إنتاج.

## ماذا يحدث داخليًا؟

Nest يعامل المصنع كمزوّد غير متزامن: يحل `inject`، ينتظر الناتج، ثم يمرّره إلى تسجيل الوحدة الديناميكية. إن رمى `getOrThrow` أو رمى المخطط قبل ذلك، يتوقف `create`. الخيارات الناتجة لقطة صحيحة بعد التحميل، لا لقطة الاستيراد.

`useClass` بديل يوثّقه TypeORM: صنف ينفّذ مصنع الخيارات. استخدمه حين تمتلئ الدالة بشرطين. للدورة، `useFactory` يكفي.

وحدات لاحقة، مثل `AuthenticationModule.forRootAsync` و `ThrottlerModule.forRootAsync`، تتبع الفكرة: أسرار وعمر رموز وحدود طلبات تأتي من `ConfigService` لا من أرقام مكتوبة في `app.module.ts`.

## مثال عملي

المكتبة تقرأ `DATABASE_URL` بعد مخطط Zod. في التطوير العنوان يشير إلى PostgreSQL محلي. في الاختبار المصنع نفسه يستقبل `ConfigService` مزيفًا يعيد عنوان قاعدة الاختبار. لا يوجد فرع `if (process.env.NODE_ENV)` داخل `BooksService`.

حين يصل فصل Prisma، احذف معاينة TypeORM إن لم تخترها، وأبقِ العادة: أي خيار اتصال يُبنى في مصنع أو في بنّاء خدمة بعد التحقق.

## أخطاء شائعة

- قراءة `process.env` داخل `useFactory` رغم حقن `ConfigService`، فتفوت قيم المخطط المحوّلة.
- `synchronize: true` منسوخًا من مثال سريع.
- نسيان `inject` مع وجود وسيط في المصنع، فيصل `config` كـ `undefined`.
- استدعاء `forRoot` و `forRootAsync` للاتصال نفسه في وحدتين.

## متى نستخدمه؟

حين تُبنى خيارات وحدة من مزوّد يُحل أثناء التهيئة: قاعدة، بريد، حد معدّل الطلبات، سر توقيع.

## متى لا نستخدمه؟

لقيمة حرفية لا تعتمد على البيئة، مثل `GreetingModule.forRoot("مرحبًا")` من فصل الوحدات. `forRoot` المباشر أوضح.

## مقارنة

| `forRoot(options)` | `forRootAsync({ useFactory, inject })` |
| --- | --- |
| الكائن يُحسب عند الاستيراد | المصنع يُحسب عند التهيئة |
| مناسب لحرفي ثابت | مناسب لـ `ConfigService` |
| لا حقن | `inject` و `imports` |

## تمارين

### تمرين سهل

لماذا قد يكون `process.env.DATABASE_URL` فارغًا داخل كائن `forRoot` رغم صحة `.env`؟

<details>
<summary>الحل</summary>

الكائن يُقيَّم عند استيراد الملف، وقد يسبق تحميل dotenv داخل `ConfigModule`. المصنع في `forRootAsync` يعمل بعد ذلك التحميل.

</details>

### تمرين متوسط

ماذا تغيّر في المثال حتى يقرأ المصنع كائن نطاق `database` فيه `url`؟

<details>
<summary>الحل</summary>

`TypeOrmModule.forRootAsync(databaseConfig.asProvider())` إن كان شكل الكائن مطابقًا، أو مصنع يحقن `databaseConfig.KEY` ويعيد `{ type: "postgres", url: db.url, synchronize: false, autoLoadEntities: true }`.

</details>

### تمرين متقدم

اختباران ينشئان تطبيقين. لماذا حقن `ConfigService` في المصنع أنظف من كتابة العنوان في أعلى `app.module.ts`؟

<details>
<summary>الحل</summary>

كل `TestingModule` يقدّم `ConfigService` بقيمة مختلفة. أعلى الملف لقطة واحدة على العملية. المصنع يعيد الحساب لكل تطبيق عند تهيئته.

</details>

## أسئلة مراجعة

- ما الفرق في توقيت `forRoot` و `forRootAsync`؟
- لماذا `synchronize: false` في معاينة القاعدة؟
- ماذا يفعل `databaseConfig.asProvider()`؟

## الخلاصة

`forRootAsync` أول جسر حقيقي من الإعداد إلى وحدة تحتاج عنوانًا أو سرًا عند الإقلاع. اقرأ عبر `ConfigService` بعد التحقق، وأبقِ أسرار الاتصال خارج الكود.

## ماذا بعد؟

[أمثلة عملية](./examples.md) تجمع ملف البيئة والمخطط وقراءة المنفذ في مكتبة واحدة.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
- NestJS — TypeORM: https://docs.nestjs.com/data/typeorm
