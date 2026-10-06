# مقدمة Dynamic Modules

## الهدف من هذا الدرس

تميّز الوحدة الساكنة عن الوحدة التي تُبنى بياناتها وقت الإقلاع عبر `forRoot` أو `register`، من غير بناء مكتبة بعد.

## المشكلة التي يحلها

`@Module({ imports, providers })` ثابت في الكود. وحدة قاعدة البيانات تحتاج عنوان الاتصال من الخارج. إن كتبت العنوان داخل الملف صارت الوحدة غير قابلة لإعادة الاستخدام. الوحدة الديناميكية دالة تُرجع `@Module` بعد أن تستلم الخيارات.

## الفكرة ببساطة

الصنف ما زال وحدة. الدالة `forRoot` تُرجع كائنًا يصف الوحدة ومعها المزوّدات التي صُنعت من الخيارات. من يستورد يستدعي الدالة مرة في مكان التكوين.

## التعريف التقني

Dynamic Module كائن له شكل بيانات `@Module` زائد المفتاح `module` الذي يشير إلى الصنف. التوثيق يعرّف دوالًا معتادة: `forRoot` و `forRootAsync` لإعداد مرة على مستوى التطبيق، و `register` و `registerAsync` لإعداد قد يختلف عند كل استيراد.

`forRootAsync` تنتظر مصنعًا أو حقنًا قبل بناء الخيارات، مثل قراءة `ConfigService`. الخصائص الثابتة على `@Module` تُدمَج مع ما ترجعه الدالة.

من يعيد تصدير الوحدة بعد التكوين يصدّر الصنف ولا يستدعي `forRoot` من جديد.

هذا الدرس مدخل. `ConfigurableModuleBuilder` و `ModuleRef` والتحميل الكسول في فصل Dynamic Modules المتقدم.

## Mental Model

```text
AppModule
  imports: [DataModule.forRoot({ url })]
       |
       دالة تُرجع { module: DataModule, providers, exports }
ميزات أخرى
  imports: [DataModule]   // الصنف فقط، بعد أن كُوّنت مرة
```

## أبسط مثال

```ts
import { DynamicModule, Module } from "@nestjs/common";

@Module({})
export class GreetingModule {
  static forRoot(message: string): DynamicModule {
    return {
      module: GreetingModule,
      providers: [
        {
          provide: "GREETING",
          useValue: message,
        },
      ],
      exports: ["GREETING"],
    };
  }
}
```

```ts
@Module({
  imports: [GreetingModule.forRoot("مرحبًا")],
})
export class AppModule {}
```

## شرح المثال

`GreetingModule` صنف فارغ تقريبًا على `@Module({})`. التكوين الحقيقي يرجع من `forRoot`. المفتاح `module` يخبر Nest أي صنف يمثّل هذا الوصف.

`"GREETING"` رمز حقن. القيمة النص تمر من مستدعي `forRoot` إلى من يحقن الرمز. التصدير يجعل الرمز مرئيًا خارج الوحدة إن لزم.

## ماذا يحدث داخليًا؟

Nest يستدعي الدالة أثناء قراءة `imports`. الناتج Dynamic Module يُعامل كبيانات وحدة: مزوّداته تُسجَّل وتصديراته تُحترم كأي `@Module` ساكن. لا يوجد نوع ثانٍ من الحاوية.

`useValue` هنا يضع النص كما هو. لاحقًا سترى `useClass` و `useFactory` عندما يصير الخيار كائن اتصال لا نصًا.

إن استوردت `GreetingModule` من غير `forRoot` فلن يُسجَّل `"GREETING"` لأن بيانات الزينة الساكنة فارغة. هذا فرق عملي عن الوحدة التي تكتب مزوّداتها مباشرة فوق الصنف.

## مثال عملي

`TypeOrmModule.forRoot({ url })` أو `ConfigModule.forRoot()` في الجذر يتبعان هذا الشكل: خيار خارجي يصنع مزوّدات. وحدات الميزات تستورد `TypeOrmModule.forFeature([...])` أو تعيد استخدام ما صُدّر، حسب المكتبة. الفكرة واحدة حتى لو اختلف اسم الدالة.

## أخطاء شائعة

- استدعاء `forRoot` في كل Feature Module فيتكرر التكوين.
- نسيان المفتاح `module` في الكائن المُرجَع.
- توقع أن خصائص `@Module` الساكنة تُستبدَل كليًا. التوثيق يقول إنها تُدمَج مع الناتج.
- بناء Dynamic Module الآن لكل ميزة في المتجر. الأكثر وحدة ساكنة حتى تظهر خيارات حقًا.

## متى نستخدمه؟

عندما تكتب مكتبة أو وحدة داخلية لا تعرف خياراتها إلا عند تركيب التطبيق: عنوان، سر، اسم طابور.

## متى لا نستخدمه؟

لوحدة كتب عناوينها معروفة داخل التطبيق نفسه. `@Module` ساكن أوضح. أرجئ `forRootAsync` حتى درس الإعداد.

## مقارنة

| | وحدة ساكنة | Dynamic Module |
| --- | --- | --- |
| أين البيانات | فوق الصنف | كائن تُرجعه دالة |
| الخيارات | ثابتة في الكود | يمررها المستدعي |
| إعادة الاستخدام كحزمة | أضعف إن كان فيها سر المشروع | هذا هو سبب النمط |

مقابل إعادة التصدير: إعادة التصدير تمرر وحدة موجودة. الديناميكي يصنع بيانات الوحدة من وسيط.

## تمارين

### تمرين سهل

ماذا يمثل المفتاح `module` داخل الكائن الذي تُرجعه `forRoot`؟

<details>
<summary>الحل</summary>

صنف الوحدة الذي يمثّله هذا الوصف، مثل `GreetingModule`.

</details>

### تمرين متوسط

لماذا تستدعي الميزة التي تريد فقط استخدام المزوّد المُعد مسبقًا الصنف المعاد تصديره بدل `DataModule.forRoot(options)` مرة ثانية؟

<details>
<summary>الحل</summary>

`forRoot` مكان التكوين. الاستدعاء الثاني يعيد البناء. بعد التكوين يكفي استيراد الصنف أو إعادة تصديره.

</details>

### تمرين متقدم

ارسم توقيع `forRoot` لوحدة `CacheModule` تأخذ `{ ttl: number }` وتصدّر رمز `"CACHE_TTL"`. لا تكتب مصنعًا غير متزامن بعد.

<details>
<summary>الحل</summary>

```ts
static forRoot(options: { ttl: number }): DynamicModule {
  return {
    module: CacheModule,
    providers: [{ provide: "CACHE_TTL", useValue: options.ttl }],
    exports: ["CACHE_TTL"],
  };
}
```

</details>

## أسئلة مراجعة

- ما الفرق بين بيانات `@Module` الساكنة و Dynamic Module؟
- متى تختار `forRoot` ومتى يكفي `@Module` عادي؟
- لماذا لا يُستدعى `forRoot` عند كل مستورد؟

## الخلاصة

الوحدة الديناميكية دالة تُرجع وصف وحدة مبنيًا من خيارات. `module` يربط الوصف بالصنف. التكوين مرة في الجذر، والميزات تستهلك الناتج. التفصيل الأكبر مؤجَّل حتى تبني وحدات ساكنة بثقة.

## ماذا بعد؟

فصل الوحدات اكتمل بما يكفي لتركيب التطبيق. الفصل التالي هو Controllers: أين يُعلَن مسار HTTP، ولماذا تبقى القاعدة خارج المتحكم.

## المصادر الرسمية

- [Dynamic modules](https://docs.nestjs.com/fundamentals/dynamic-modules)
- [Modules](https://docs.nestjs.com/modules)
