# Module ثابت مقابل Dynamic Module

## الهدف من هذا الدرس

تمييز الوحدة التي تُكتب بياناتها كلها في `@Module` عن الوحدة التي تُبنى بياناتها عند الاستيراد عبر دالة ترجع `DynamicModule`.

## المشكلة التي يحلها

`IdGeneratorModule` الثابت لا يستطيع أن يعطي `UsersModule` سابقة `usr_` و `OrdersModule` سابقة `ord_` من غير تكرار الملفات. المستورد لا يؤثر في مزودات الوحدة المضيفة ما دام الربط ساكنًا. الوحدة الديناميكية واجهة استيراد تُخصص السلوك وقت الإقلاع.

## الفكرة ببساطة

الصنف يبقى وحدة. دالة ثابتة مثل `register` أو `forRoot` ترجع كائنًا فيه المفتاح `module` وبقية حقول `@Module`. هذا الكائن يوضع في `imports` كما توضع الوحدة الساكنة. الفرق أن الحقول جاءت من كود حسب خيار المستورد، لا من زخرفة ثابتة فقط.

## التعريف التقني

الربط الساكن: `providers` و `exports` و `imports` على `@Module`. Nest يبني المضيف، ثم يجعل المصدَّر مرئيًا للمستورد، ثم يحقن.

`DynamicModule` من `@nestjs/common` له شكل metadata الوحدة زائد `module` الذي يشير إلى الصنف. كل الحقول اختيارية إلا `module`. الدالة اصطلاحًا `register` أو `forRoot` أو `forFeature`، ويمكن أي اسم، ويجب أن تكون `static` لأنها تُستدعى على الصنف لا على نسخة.

`imports` يقبل صنف وحدة أو القيمة الراجعة من الدالة. الوحدة الديناميكية تستطيع أن تستورد وحدات أخرى في حقل `imports` الراجع.

زخرفة `@Module` على الصنف تُدمَج مع الكائن الراجع. صنف فارغ `@Module({})` شائع عندما يأتي كل التسجيل من الدالة.

من يعيد التصدير بعد التكوين يضع الصنف في `exports` ولا يستدعي `forRoot` أو `register` مرة ثانية. الاستدعاء الثاني يعيد البناء. فصل الوحدات مهّد لهذا، وهنا يصبح قاعدة المكتبة.

## Mental Model

```text
ساكن:     imports: [UsersModule]
ديناميكي: imports: [IdGeneratorModule.register({ prefix: 'usr' })]
الكائن الراجع: { module: IdGeneratorModule, providers, exports }
إعادة التصدير: exports: [IdGeneratorModule]   بلا استدعاء ثانٍ
```

الدالة مكان الخيار. الصنف مكان الهوية.

## أبسط مثال

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { IdGenerator } from './id-generator.js';

@Module({})
export class IdGeneratorModule {
  static register(): DynamicModule {
    return {
      module: IdGeneratorModule,
      providers: [IdGenerator],
      exports: [IdGenerator],
    };
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { IdGeneratorModule } from '../id-generator/id-generator.module.js';

@Module({
  imports: [IdGeneratorModule.register()],
})
export class UsersModule {}
```

## شرح المثال

`register` بلا خيار بعد. الشكل هو المقصود: المستورد يستدعي الدالة، والدالة تصرّح `module`. استيراد `IdGeneratorModule` من غير النداء لا يسجل `IdGenerator` إن كانت الزخرفة فارغة. هذا أوضح فرق عملي عن الوحدة الساكنة التي تكتب المزود فوق الصنف.

## ماذا يحدث داخليًا؟

1. تقييم الملف يعرّف الصنف والدالة.
2. بناء `UsersModule` يستدعي `register()` فور قراءة `imports`.
3. الإطار يرى كائن `DynamicModule` ويدمج الحقول مع ما على `@Module`.
4. يسجل `IdGenerator` ويصدّره داخل حدود هذا الاستيراد.
5. استيراد آخر يستدعي `register()` ينتج كائنًا آخر. إن كان الخيار مختلفًا فالتسجيل مختلف، وهذا موضوع `register` مقابل `forRoot`.
6. إعادة تصدير الصنف بعد استدعاء واحد تمرر ما بُني، ولا تشغّل الدالة من جديد.

## مثال عملي

`ConfigModule.forRoot()` و `TypeOrmModule.forRoot({ url })` في الجذر يتبعان الشكل: خيار خارجي يصنع مزودات. ميزات الكتب تستورد ما صُدّر أو `forFeature` حسب المكتبة، ولا تعيد `forRoot`. احفظ القاعدة قبل أن تكتب مكتبتك: مكان واحد يستدعي دالة التكوين، والباقي يستورد الصنف المعاد تصديره.

## أخطاء شائعة

- استدعاء `forRoot` في كل ميزة فيتكرر الاتصال.
- نسيان المفتاح `module`.
- توقع أن استيراد الصنف وحده يكفي عندما الزخرفة فارغة والدالة هي التي تسجل المزودات.
- وضع كائن عادي في `imports` بلا `module`.
- إعادة التصدير عبر `exports: [DatabaseModule.forRoot(url)]` من وحدة لا تملك الخيار.

## متى نستخدمه؟

مكتبة أو وحدة داخلية تحتاج خيارًا من المستورد: سابقة، عنوان، مهلة.

## متى لا نستخدمه؟

ميزة كتب تعرف مزوداتها داخل التطبيق. `@Module` الساكن أوضح. لا تبنِ `forRoot` لخيار لا يتغير ولا يأتي من الخارج.

## مقارنة

| | ساكن | ديناميكي |
| --- | --- | --- |
| مكان البيانات | `@Module` | كائن راجع من دالة ثابتة |
| خيار المستورد | لا يغيّر المضيف | يدخل عبر معاملات الدالة |
| المفتاح الزائد | — | `module` |
| إعادة التصدير | الصنف | الصنف بعد استدعاء التكوين مرة |

## تمارين

### تمرين سهل

ما الحقل الإلزامي في الكائن الذي ترجع الدالة؟

<details>
<summary>الحل</summary>

`module`، ويشير إلى صنف الوحدة.

</details>

### تمرين متوسط

لماذا استيراد `IdGeneratorModule` من غير `register()` لا يسجل المولّد إن كانت الزخرفة `@Module({})`؟

<details>
<summary>الحل</summary>

التسجيل مكتوب في الكائن الراجع من الدالة. الزخرفة فارغة، فالصنف وحده بلا نداء لا يضيف `IdGenerator`.

</details>

### تمرين متقدم

`CoreModule` استدعى `DatabaseModule.forRoot(url)`. ميزة الكتب تريد العميل. ماذا تضع في `exports`، وماذا لا تستدعي؟

<details>
<summary>الحل</summary>

`exports: [DatabaseModule]`. لا تستدعي `forRoot` مرة ثانية. التكوين تم في النواة.

</details>

## أسئلة مراجعة

- ماذا يضيف `DynamicModule` على شكل `@Module`؟
- لماذا الدالة `static`؟
- أين يعيش استدعاء التكوين في التطبيق؟

## الخلاصة

الوحدة الساكنة بياناتها ثابتة على الصنف. الديناميكية ترجع الوصف عند الاستيراد حتى يمر الخيار. أعد تصدير الصنف بعد استدعاء واحد.

## ماذا بعد؟

`forRoot`: تكوين مرة للتطبيق، غالبًا مع `global`.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
