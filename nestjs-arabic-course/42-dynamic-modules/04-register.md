# register

## الهدف من هذا الدرس

تمرير خيار مختلف في كل استيراد عبر `register`، بحيث تشارك الميزات المولّد نفسه وتختلف السابقة أو المهلة.

## المشكلة التي يحلها

`forRoot` يثبت خيارًا واحدًا للتطبيق. سابقة معرف المستخدم `usr` وسابقة الطلب `ord` خياران شرعيان في الإقلاع نفسه. نسخ الملفات يكسر التحديث. `register({ prefix })` يعيد `DynamicModule` لكل نداء، والتوثيق يبني مثال `IdGeneratorModule` على هذا الاسم اصطلاحًا عندما يختلف التكوين لكل مستهلك.

## الفكرة ببساطة

الدالة الثابتة تقبل كائن خيار وتضعه في مزود، غالبًا برمز، وتصدّر الخدمة التي تقرأ الرمز. `UsersModule` ينادي `register({ prefix: 'usr' })` و `OrdersModule` ينادي `register({ prefix: 'ord' })`. كل نداء وصف وحدة. احذر مشاركة حالة على Singleton عام إن كان الخيار يجب أن يبقى مع الميزة التي طلبته.

## التعريف التقني

من دليل التوثيق: المستورد يكتب `IdGeneratorModule.register({ prefix: 'usr' })`. الدالة ترجع `DynamicModule`. الخدمة تأخذ الخيار بالحقن لا بثابت داخل الملف.

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { IdGenerator } from './id-generator.js';

export interface IdGeneratorOptions {
  prefix: string;
}

@Module({})
export class IdGeneratorModule {
  static register(options: IdGeneratorOptions): DynamicModule {
    return {
      module: IdGeneratorModule,
      providers: [
        { provide: 'ID_OPTIONS', useValue: options },
        IdGenerator,
      ],
      exports: [IdGenerator],
    };
  }
}
```

```ts
import { Inject, Injectable } from '@nestjs/common';
import { randomUUID } from 'node:crypto';

@Injectable()
export class IdGenerator {
  constructor(@Inject('ID_OPTIONS') private readonly options: { prefix: string }) {}

  next(): string {
    return `${this.options.prefix}_${randomUUID()}`;
  }
}
```

إن سجلت النداءان المزود بالرمز نفسه في نطاق يتصادم، فالأخير قد يغلب حسب طريقة التسجيل. لعزل حقيقي بين ميزتين اجعل الرمز فريدًا لكل نداء أو احقن الخدمة في الوحدة المستهلكة فقط من غير تصدير عام. المثال الرسمي يصدّر `IdGenerator` كي يحقنه `UsersService` في تلك الوحدة. أبقِ الوحدة غير عامة حتى لا يختلط خيار `usr` مع `ord` في حاوية واحدة بالرمز `'ID_OPTIONS'`.

## Mental Model

```text
UsersModule  -> register({ prefix: 'usr' }) -> مولّد يراه مستخدمو هذه الوحدة
OrdersModule -> register({ prefix: 'ord' }) -> مولّد يراه الطلبات
لا global
لا forRoot ثانٍ يحمل السابقات كلها إن كانت السابقة شأن الميزة
```

اسم الدالة اصطلاح. السلوك هو كائن لكل استدعاء.

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { IdGeneratorModule } from '../id-generator/id-generator.module.js';
import { UsersService } from './users.service.js';

@Module({
  imports: [IdGeneratorModule.register({ prefix: 'usr' })],
  providers: [UsersService],
})
export class UsersModule {}
```

`UsersService` تحقن `IdGenerator` وتستدعي `next()`. الناتج يبدأ بـ `usr_`.

## شرح المثال

`useValue: options` يلتقط الكائن الذي مرره هذا الاستيراد. لأن الوحدة ليست `global`، التصدير مرئي لمن استورد هذا الكائن الراجع. ميزة الطلبات تكرر النمط بـ `ord`. لا تعِد تصدير `IdGeneratorModule` من الجذر بعد `register` واحد ثم تتوقع أن الميزة الثانية تغيّر السابقة؛ إعادة التصدير تشارك ما بُني.

## ماذا يحدث داخليًا؟

1. كل ظهور لـ `register` في `imports` يُستدعى أثناء بناء تلك الوحدة.
2. الإطار يسجل المزودات المرتبطة بذلك الكائن.
3. الخدمة داخل تلك الحدود ترى `'ID_OPTIONS'` الخاص بالنداء إن بقي التسجيل محليًا.
4. جعل الوحدة عامة مع رمز نصي ثابت يخلط الخيارات. لذلك `register` المتعدد يبقى غير عام، أو يستعمل رمزًا فريدًا لكل تكوين.
5. الدمج مع `@Module` الثابت ما زال ساريًا.

## مثال عملي

مهلة كاش لكل ميزة: `CacheModule.register({ ttlSeconds: 5 })` في البحث، و `register({ ttlSeconds: 60 })` في صفحة الكتاب. الخدمتان تحقن `CacheClient` المصدَّر من استيرادهما. اختبار كل وحدة يمرر مهلتها في `imports` من غير التأثير على الأخرى.

## أخطاء شائعة

- `global: true` مع `register` متعدد ورمز ثابت.
- تسمية الدالة `forRoot` ثم استدعاؤها في كل ميزة، فيضل القارئ.
- نسيان `exports` فلا يرى `UsersService` المولّد.
- مشاركة كائن خيارات قابل للتغيير بين النداءات. مرر كائنًا لكل نداء.
- إعادة تصدير الصنف من الجذر واعتبارها تكوينًا لكل الميزة.

## متى نستخدمه؟

خيار يصف استهلاك هذه الوحدة: سابقة، مهلة، اسم طابور للميزة.

## متى لا نستخدمه؟

اتصال قاعدة واحد. ذلك `forRoot` في الجذر. خيار من `ConfigService` لكل ميزة ينتظر `registerAsync`.

## مقارنة

| | `forRoot` | `register` |
| --- | --- | --- |
| من يستدعي | الجذر عادة | كل مستهلك |
| الخيار | واحد للتطبيق | حسب الاستيراد |
| `global` | شائع | يفسد تعدد الخيارات إن كان الرمز مشتركًا |
| إعادة التصدير | الصنف بعد النداء الوحيد | لا تُعمم نداء ميزة على الباقي |

## تمارين

### تمرين سهل

أي اسم دالة يوضحه التوثيق لسابقة مختلفة لكل كيان؟

<details>
<summary>الحل</summary>

`register`، كما في `IdGeneratorModule.register({ prefix: 'usr' })`.

</details>

### تمرين متوسط

لماذا إبقاء `IdGeneratorModule` غير عامة في مثال السابقات؟

<details>
<summary>الحل</summary>

حتى لا يختلط خيار `usr` مع `ord` على رمز واحد مرئي للتطبيق كله.

</details>

### تمرين متقدم

ميزة أعادت تصدير `IdGeneratorModule` بعد `register({ prefix: 'usr' })`. ماذا ترى ميزة تستورد الأولى وتريد `ord`؟

<details>
<summary>الحل</summary>

ترى ما بُني عند النداء الأول، أي سابقة `usr`، إن كانت تستهلك المصدَّر كما هو. للحصول على `ord` تستدعي `register` في حدودها، ولا تعتمد على إعادة تصدير نداء غيرها.

</details>

## أسئلة مراجعة

- ماذا يرجع `register`؟
- أين يُخزن `prefix` حتى تقرأه الخدمة؟
- متى يضر `global` مع هذه الدالة؟

## الخلاصة

`register` تكوين عند كل استيراد. اجعل الخيار محليًا لتلك الوحدة، واترك `forRoot` للإعداد الواحد في الجذر.

## ماذا بعد؟

`registerAsync`: المصنع نفسه عندما يأتي الخيار من مزود آخر.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
