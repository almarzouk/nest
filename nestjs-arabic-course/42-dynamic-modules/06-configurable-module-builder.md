# ConfigurableModuleBuilder

## الهدف من هذا الدرس

توليد `register` و `registerAsync` (أو `forRoot` و `forRootAsync`) ورمز الخيارات من `ConfigurableModuleBuilder` بدل كتابة المزودات في كل مكتبة.

## المشكلة التي يحلها

نسخ `forRootAsync` يدويًا يخطئ في `inject` أو ينسى الدمج مع `global`. البنّاء في `@nestjs/common` ينتج صنفًا أساسيًا ودوالًا ورمز `MODULE_OPTIONS_TOKEN`. وحدتك تمتد الصنف وتضيف الخدمة. المستورد يستدعي الدالة كما في الدروس السابقة.

## الفكرة ببساطة

في ملف تعريف تنشئ البنّاء على نوع الخيارات وتستدعي `build()`. تصدر `ConfigurableModuleClass` و `MODULE_OPTIONS_TOKEN`. صنف الوحدة يمتد الصنف المولَّد. الخدمة تحقن الرمز. `setClassMethodName('forRoot')` يبدل الاسم الاصطلاحي. `setExtras` يفصل علم `isGlobal` عن الخيارات التي تراها الخدمة. `setFactoryMethodName` يبدل `create`.

## التعريف التقني

```ts
import { ConfigurableModuleBuilder } from '@nestjs/common';

export interface IdGeneratorModuleOptions {
  prefix: string;
}

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<IdGeneratorModuleOptions>().build();
```

```ts
import { Module } from '@nestjs/common';
import { IdGenerator } from './id-generator.js';
import { ConfigurableModuleClass } from './id-generator.module-definition.js';

@Module({
  providers: [IdGenerator],
  exports: [IdGenerator],
})
export class IdGeneratorModule extends ConfigurableModuleClass {}
```

المستورد: `IdGeneratorModule.register({ prefix: 'usr' })` أو `registerAsync`. الاسم الافتراضي `register`.

لتوليد `forRoot`:

```ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<IdGeneratorModuleOptions>()
    .setClassMethodName('forRoot')
    .build();
```

`setExtras` لعلم لا يخص الخدمة:

```ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<IdGeneratorModuleOptions>()
    .setExtras({ isGlobal: false }, (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,
    }))
    .build();
```

`isGlobal` يصل إلى `global` على الوحدة ولا يظهر داخل كائن `MODULE_OPTIONS_TOKEN`. الخدمة تحقن الرمز وترى `prefix` فقط.

`OPTIONS_TYPE` و `ASYNC_OPTIONS_TYPE` من ناتج `build()` لأنواع توسيع الدوال إن أعدت كتابة `register` وناديت `super.register(options)`.

## Mental Model

```text
module-definition.ts  -> build() -> دوال + رمز
IdGeneratorModule     -> extends + providers
الخدمة                -> @Inject(MODULE_OPTIONS_TOKEN)
المستورد              -> register أو forRoot حسب الاسم الذي ضبطته
```

`setExtras` بوابة خصائص الوحدة. الرمز بوابة خصائص الخدمة.

## أبسط مثال

الملفان أعلاه ثم:

```ts
import { Inject, Injectable } from '@nestjs/common';
import { MODULE_OPTIONS_TOKEN } from './id-generator.module-definition.js';
import type { IdGeneratorModuleOptions } from './id-generator.module-definition.js';

@Injectable()
export class IdGenerator {
  constructor(
    @Inject(MODULE_OPTIONS_TOKEN)
    private readonly options: IdGeneratorModuleOptions,
  ) {}

  next(): string {
    return `${this.options.prefix}_id`;
  }
}
```

## شرح المثال

الرمز مولَّد مع الدوال، فلن تخطئ في سلسلة `'ID_OPTIONS'` بين الملفَين. `@Module` على الصنف الابن يدمج المزودات مع ما ترجعه الدالة المولَّدة. المستورد لا يرى البنّاء، يرى `register` فقط.

## ماذا يحدث داخليًا؟

1. `build()` يجهز دوالًا ثابتة على الصنف الأساسي.
2. الابن يرثها، فـ `IdGeneratorModule.register` يعمل.
3. النداء يسجل `MODULE_OPTIONS_TOKEN` بقيمة الخيار.
4. `IdGenerator` يُحقن من الزخرفة الثابتة ويرى الرمز.
5. `setExtras` يعدّل تعريف الوحدة قبل التسجيل، مثل وضع `global`.
6. إعادة التصدير تبقى على الصنف من غير نداء ثانٍ، كما في أي وحدة ديناميكية.

## مثال عملي

مكتبة كاش داخل الشركة: `setClassMethodName('forRoot')` لأن المهلة واحدة للتطبيق، و `setExtras` حتى يكتب الجذر `forRoot({ isGlobal: true, ttlSeconds: 30 })`. الخدمة تقرأ `ttlSeconds` من الرمز ولا تعرف إن كانت الوحدة عامة. ميزة تريد تسجيلًا متعددًا تترك الاسم `register` ولا تضع `isGlobal`.

## أخطاء شائعة

- حقن `'MODULE_OPTIONS_TOKEN'` كنص. الرمز هو الثابت المصدَّر.
- توقع ظهور `isGlobal` داخل الخدمة بعد `setExtras`. التوثيق يخرجه من كائن الرمز.
- نسيان `extends ConfigurableModuleClass` فتختفي `register`.
- كتابة `forRoot` يدويًا فوق البنّاء الذي ما زال يولد `register` من غير `setClassMethodName`.
- وضع الخيار السري داخل `setExtras` حيث لا تراه الخدمة التي تحتاجه.

## متى نستخدمه؟

وحدة ستعيش كواجهة استيراد ويهمها `register` و `registerAsync` بلا نسخ لصق.

## متى لا نستخدمه؟

نداء واحد داخلي لن يتكرر. دالة `forRoot` مكتوبة يدويًا في عشرين سطرًا أوضح من ملف تعريف إضافي.

## مقارنة

| الضبط | الأثر |
| --- | --- |
| `build()` الافتراضي | `register` و `registerAsync` |
| `setClassMethodName('forRoot')` | `forRoot` و `forRootAsync` |
| `setFactoryMethodName` | اسم بدل `create` في `useClass` |
| `setExtras` | خصائص تعريف الوحدة خارج رمز الخدمة |

## تمارين

### تمرين سهل

ماذا تحقن الخدمة لقراءة `prefix` بعد `build()`؟

<details>
<summary>الحل</summary>

`MODULE_OPTIONS_TOKEN` المصدَّر من ملف التعريف.

</details>

### تمرين متوسط

كيف تحصل على `forRoot` بدل `register`؟

<details>
<summary>الحل</summary>

`.setClassMethodName('forRoot')` قبل `build()`.

</details>

### تمرين متقدم

لماذا `isGlobal` في `setExtras` لا يُحقن مع `prefix`؟

<details>
<summary>الحل</summary>

لأن الدالة الثانية في `setExtras` تضعه على تعريف الوحدة (`global`) وتبقيه خارج كائن `MODULE_OPTIONS_TOKEN` الذي تراه الخدمة.

</details>

## أسئلة مراجعة

- ماذا يرث صنف الوحدة؟
- أين يُضبط اسم الدالة؟
- ماذا تفعل عند إعادة التصدير؟

## الخلاصة

البنّاء يولد دوال التكوين ورمز الخيارات. اضبط الاسم والإضافات في ملف التعريف، وحقن الرمز في الخدمة، واستدعِ الدالة مرة بالطريقة التي يناسبها `forRoot` أو `register`.

## ماذا بعد؟

`ModuleRef`: جلب مزود بالرمز بعد الإقلاع، بما في ذلك ما استخدمته لفك طرف دورة.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
