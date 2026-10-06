# forRootAsync

## الهدف من هذا الدرس

بناء خيارات `forRoot` بعد أن تتوفر اعتماديات أخرى، خصوصًا `ConfigService`، عبر مصنع قد يرجع وعدًا.

## المشكلة التي يحلها

`forRoot(process.env.DATABASE_URL)` يقرأ البيئة لحظة تقييم الزخرفة وقد يسبق التحقق والفشل الواضح. `forRootAsync` يؤخر بناء الكائن حتى يحقن الإطار المصنع بـ `ConfigService` وينتظر إن رجع `Promise`. الإقلاع يتوقف على عنوان ناقص بعد رسالة إعداد، لا على عميل بُني بسلسلة فارغة.

## الفكرة ببساطة

نفس وحدة `forRoot`، ودالة ثانية `forRootAsync` تستقبل `imports` و `useFactory` و `inject`. المصنع يعيد كائن الخيارات. الوحدة الديناميكية تسجل الخيارات كمزود ثم تبني الخدمة التي تحقنها. الاستدعاء يبقى مرة في الجذر.

## التعريف التقني

شكل المصنع كما في `registerAsync` في التوثيق، والاسم هنا `forRootAsync` لأن التكوين مرة واحدة. الحقول المتداولة:

| الحقل | الدور |
| --- | --- |
| `imports` | وحدات تُستورد حتى تُحقن مصدَّراتها في المصنع |
| `useFactory` | يعيد الخيارات أو وعدًا بها |
| `inject` | رموز تُمرر إلى المصنع بالترتيب |
| `useClass` | صنف فيه `create()` يعيد الخيارات |
| `useExisting` | يعيد استخدام صنف موجود بدل نسخة جديدة |

اختر واحدًا من `useFactory` و `useClass` و `useExisting`.

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { DatabaseService } from './database.service.js';

@Module({})
export class DatabaseModule {
  static forRootAsync(options: {
    imports?: DynamicModule['imports'];
    useFactory: (...args: never[]) => Promise<{ url: string }> | { url: string };
    inject?: never[];
  }): DynamicModule {
    return {
      module: DatabaseModule,
      imports: options.imports ?? [],
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useFactory: options.useFactory,
          inject: options.inject ?? [],
        },
        DatabaseService,
      ],
      exports: [DatabaseService],
    };
  }
}
```

الجذر:

```ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { DatabaseModule } from './database/database.module.js';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    DatabaseModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        url: config.getOrThrow<string>('DATABASE_URL'),
      }),
    }),
  ],
})
export class AppModule {}
```

`ConfigurableModuleBuilder` يولّد هذا الشكل لاحقًا. هنا الكتابة يدوية حتى ترى المزود.

## Mental Model

```text
ConfigModule يُهيأ
  -> useFactory(config) -> { url }
  -> الرمز DATABASE_OPTIONS
  -> DatabaseService تُبنى
  -> listen
```

المصنع جزء من الإقلاع. الفشل فيه يمنع الباب.

## أبسط مثال

مقطعا الوحدة والجذر أعلاه. `DatabaseService` تحقن `'DATABASE_OPTIONS'` وتقرأ `url`. لا تقرأ `process.env` داخل الخدمة إن مررت الخيار من المصنع، حتى يبقى الاختبار قادرًا على استبدال الرمز.

## شرح المثال

`imports: [ConfigModule]` يجعل `ConfigService` مرئيًا لمصنع هذه الوحدة الديناميكية. `inject` يطابق معاملات المصنع ترتيبًا. `getOrThrow` يرمي إن غاب المتغير، فيقف الإقلاع. `ConfigModule.forRoot` نفسه استدعاء تكوين مرة واحدة، وهذا اتساق مع درس `forRoot`.

## ماذا يحدث داخليًا؟

1. الإطار يستورد ما في `imports` الخاص بـ `forRootAsync`.
2. يحل رموز `inject`.
3. يستدعي المصنع وينتظر الوعد إن وُجد.
4. يسجل الناتج تحت `'DATABASE_OPTIONS'`.
5. يبني `DatabaseService` بحقن ذلك الرمز.
6. الميزات لا تعيد `forRootAsync`. من يحتاج الخدمة يستورد الوحدة العامة أو الصنف المعاد تصديره.

## مثال عملي

مهلة بريد تُقرأ من الإعداد:

```ts
useFactory: (config: ConfigService) => ({
  url: config.getOrThrow<string>('SMTP_URL'),
  timeoutMs: Number(config.get('SMTP_TIMEOUT_MS') ?? 3000),
}),
```

اختبار يمرر مصنعًا يعيد `{ url: 'smtp://example', timeoutMs: 10 }` من غير ملف `.env`. هذا ربح `forRootAsync` على قراءة البيئة داخل الـ `constructor`.

## أخطاء شائعة

- نسيان `imports` ثم تعجب أن `ConfigService` غير موجود في المصنع.
- اختلاف ترتيب `inject` عن معاملات الدالة.
- استدعاء `forRoot` و `forRootAsync` معًا على الوحدة نفسها في الجذر.
- مصنع يرجع الخيار ثم الخدمة تقرأ `process.env` فتلغي الاختبار.
- `useFactory` و `useClass` معًا. الحقول متنافية.

## متى نستخدمه؟

خيار جذر يعتمد على مزود آخر أو على عملية غير متزامنة قبل بناء العميل.

## متى لا نستخدمه؟

خيار ثابت في اختبار صغير. `forRoot({ url })` أقصر. خيار لكل ميزة ينتظر `registerAsync` لا اسم `forRoot`.

## مقارنة

| | `forRoot` | `forRootAsync` |
| --- | --- | --- |
| مصدر الخيار | معامل مباشر | مصنع أو صنف `create` |
| انتظار وعد | لا داخل التوقيع المتزامن | نعم إن رجع المصنع وعدًا |
| `ConfigService` | تقرأه أنت قبل النداء | يُحقن في المصنع |
| عدد المرات | واحد | واحد |

## تمارين

### تمرين سهل

ماذا يجب أن يطابق ترتيب `inject`؟

<details>
<summary>الحل</summary>

ترتيب معاملات `useFactory`.

</details>

### تمرين متوسط

لماذا `imports: [ConfigModule]` داخل `forRootAsync` رغم أن الجذر استورد الإعداد؟

<details>
<summary>الحل</summary>

مصنع الوحدة الديناميكية يرى مزودات ما يستورده هو. الاستيراد في الجذر لا يفتح `ConfigService` تلقائيًا داخل كائن الوحدة الراجعة إلا إن جعلت الإعداد عامًا وما زلت تحتاج التصريح حسب المكتبة. التصريح في `imports` الخاص بالنداء هو العقد الواضح.

</details>

### تمرين متقدم

اختبار يريد عنوانًا مزيفًا من غير `ConfigModule`. أي شكل تستعمل؟

<details>
<summary>الحل</summary>

`forRoot({ url: 'postgresql://test/db' })` أو `forRootAsync` بمصنع بلا `inject` يرجع الكائن مباشرة. لا حاجة لقراءة بيئة الاختبار.

</details>

## أسئلة مراجعة

- أي حقول متنافية في الاستدعاء غير المتزامن؟
- أين يبقى الاستدعاء في التطبيق؟
- ماذا يفعل الإطار بوعد المصنع؟

## الخلاصة

`forRootAsync` يبني خيار الجذر بعد حقن اعتماديات المصنع وينتظر الوعد. مرة واحدة، ثم الخدمة تستهلك الرمز لا البيئة مباشرة.

## ماذا بعد؟

`register`: التكوين نفسه بشكل قد يختلف مع كل مستورد.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
