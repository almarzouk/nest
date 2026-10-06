# forRoot

## الهدف من هذا الدرس

استخدام اسم `forRoot` عندما تُكوَّن الوحدة مرة واحدة للتطبيق، وتمرير الخيار إلى مزود عبر رمز حقن، مع إبقاء الاستدعاء في الجذر.

## المشكلة التي يحلها

عنوان قاعدة أو سابقة عامة ليست شأن كل ميزة. إن استدعت كل وحدة `forRoot` بنَت اتصالًا ثانيًا أو خالفت العنوان. الاصطلاح المجتمعي الذي يشرحه التوثيق: `forRoot` لتكوين مرة على مستوى التطبيق. الاسم ليس قيد لغة، واتباعه يجعل قارئ `AppModule` يعرف أن هذا مكان الإعداد الوحيد.

## الفكرة ببساطة

`static forRoot(options): DynamicModule` تبني المزودات من `options` وترجع `{ module, providers, exports }`. غالبًا `global: true` حتى لا تعيد الميزات الاستيراد. من غير `global`، وحدة نواة تستورد `forRoot` وتعيد تصدير الصنف.

## التعريف التقني

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { DatabaseService } from './database.service.js';

@Module({})
export class DatabaseModule {
  static forRoot(url: string): DynamicModule {
    return {
      module: DatabaseModule,
      global: true,
      providers: [
        { provide: 'DATABASE_URL', useValue: url },
        DatabaseService,
      ],
      exports: [DatabaseService],
    };
  }
}
```

```ts
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class DatabaseService {
  constructor(@Inject('DATABASE_URL') private readonly url: string) {}

  dsn(): string {
    return this.url;
  }
}
```

`AppModule` فقط يستدعي `DatabaseModule.forRoot(process.env.DATABASE_URL ?? '')`. الميزات تحقن `DatabaseService` لأن الوحدة عامة، أو تستورد الصنف المعاد تصديره إن لم تضع `global`.

`forFeature` في مكتبات مثل TypeORM شريحة بعد `forRoot`: كيانات هذه الميزة على اتصال الجذر. لا تستدعِ `forRoot` مكان `forFeature`.

زخرفة `@Module` الثابتة تُدمَج مع الراجع. إن وضعت مزودًا على الاثنين فاعرف ناتج الدمج ولا تكرر الرمز بالخطأ.

## Mental Model

```text
AppModule
  DatabaseModule.forRoot(url)  -> مرة، ربما global
BooksModule
  يحقن DatabaseService         -> بلا forRoot
```

الخيار يدخل من الجذر. الخدمة تقرأه بالرمز.

## أبسط مثال

المقطعان أعلاه. أضف في الجذر:

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module.js';

@Module({
  imports: [DatabaseModule.forRoot('postgresql://localhost/shop')],
})
export class AppModule {}
```

## شرح المثال

`useValue` يربط النص بالرمز `'DATABASE_URL'`. `DatabaseService` مزود صنف عادي داخل الكائن الراجع فيُبنى بعد توفر الرمز. `exports` يجعل الخدمة مرئية. `global: true` يلغي حاجة `imports` في كل ميزة. العنوان الثابت في المثال للتعلم؛ في التطبيق يأتي من إعداد لا من سلسلة في المصدر.

## ماذا يحدث داخليًا؟

1. تحميل `AppModule` يستدعي `forRoot` بمعامل الجذر.
2. يُسجَّل الرمز والخدمة في تلك الوحدة الديناميكية.
3. مع `global` تصبح المصدَّرات مرئية للتطبيق.
4. حقن `DatabaseService` في أي ميزة يجد النسخة نفسها `DEFAULT`.
5. استدعاء ثانٍ في ميزة أخرى يسجل تكوينًا آخر. تجنبه لـ `forRoot`.
6. إعادة التصدير من نواة تستعمل `exports: [DatabaseModule]` بعد الاستدعاء الأول فقط.

## مثال عملي

`CacheModule.forRoot({ ttlSeconds: 30 })` في الجذر يصدّر `CacheService` عامة. الكتب والطلبات تحقن الخدمة وترى المهلة نفسها. ميزة تريد مهلة مختلفة لا تستدعي `forRoot` ثانٍ. إما وسيط على الدالة `get(key, ttl)` أو وحدة `register` منفصلة للخيار المحلي، وذلك الدرس بعد `forRootAsync`.

## أخطاء شائعة

- `forRoot` داخل كل `imports` ميزة.
- نسيان تصدير الخدمة أو الرمز الذي سيُحقن خارجًا.
- `global: true` على وحدة ميزة فيها متحكمات، فيتسع السطح.
- وضع السر داخل المستودع في معامل `forRoot`. المعامل يُمرَّر من بيئة الإقلاع.
- خلط الاسم مع `register` ثم استدعاؤه مرة في الجذر ومرة في الميزة بعناوين مختلفة.

## متى نستخدمه؟

اتصال أو إعداد واحد للتطبيق: قاعدة، إعداد عام، بريد بعنوان واحد.

## متى لا نستخدمه؟

خيار يختلف لكل مستورد، مثل سابقة المعرف. ذلك `register`. خيار يُقرأ من `ConfigService` بشكل غير متزامن ينتظر `forRootAsync`.

## مقارنة

| الاسم الاصطلاحي | عدد التكوينات | مكان الاستدعاء |
| --- | --- | --- |
| `forRoot` | واحد للتطبيق | الجذر أو نواة |
| `register` | قد يختلف لكل مستورد | وحدة الميزة |
| `forFeature` | شريحة على جذر مُعد | وحدة الميزة بعد الجذر |

## تمارين

### تمرين سهل

من يستدعي `forRoot` لعنوان قاعدة واحد؟

<details>
<summary>الحل</summary>

مكان تكوين واحد، عادة `AppModule` أو وحدة نواة. الميزات لا تعيد النداء.

</details>

### تمرين متوسط

لماذا `@Inject('DATABASE_URL')` وليس حقن النص مباشرة؟

<details>
<summary>الحل</summary>

النص قيمة مسجّلة برمز. الحقن يطلب الرمز. الخدمة لا تستورد عنوان الميزة، فالجذر هو من يمرره.

</details>

### تمرين متقدم

`CoreModule` يعيد التصدير. ماذا يُمنع داخل `BooksModule`؟

<details>
<summary>الحل</summary>

`DatabaseModule.forRoot(...)` مرة أخرى. الكتب تستورد `CoreModule` أو تعتمد على `global` وتحقن الخدمة.

</details>

## أسئلة مراجعة

- ماذا يغيّر `global: true`؟
- كيف يصل `url` إلى الخدمة؟
- ما فرق الاسم عن `register` في اصطلاح التوثيق؟

## الخلاصة

`forRoot` تكوين مرة. الخيار يصبح مزودًا، والجذر يملك الاستدعاء، والميزات تحقن الناتج أو تستورد الصنف المعاد تصديره.

## ماذا بعد؟

`forRootAsync`: الخيار يأتي من `ConfigService` ووعد.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
