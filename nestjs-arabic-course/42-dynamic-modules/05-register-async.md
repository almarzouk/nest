# registerAsync

## الهدف من هذا الدرس

تمرير خيار `register` عندما يُبنى الخيار من مزود آخر، بمصنع أو صنف `create`، مع إبقاء التكوين عند المستورد لا في الجذر فقط.

## المشكلة التي يحلها

سابقة المعرف ليست ثابتة في المصدر وقد تأتي من `ConfigService` لهذه الميزة فقط: `USER_ID_PREFIX` مقابل `ORDER_ID_PREFIX`. `register({ prefix })` يحتاج القيمة الآن. `registerAsync` ينتظر المصنع كما يفعل `forRootAsync`، والفرق في الاصطلاح: النداء يبقى داخل وحدة الميزة لأن الخيار ليس إعداد التطبيق كله.

## الفكرة ببساطة

`registerAsync` يستقبل كائنًا فيه `imports` وواحد من `useFactory` أو `useClass` أو `useExisting`. المصنع يعيد خيارات هذه الوحدة. الإطار يحقن ما في `inject` بالترتيب وينتظر الوعد. الناتج يُسجل رمزًا تقرأه الخدمة.

## التعريف التقني

حقول التوثيق لـ `registerAsync`:

- `imports`: وحدات يرى المصنع أو صنف الخيارات مصدَّراتها.
- `useFactory`: يعيد الخيارات أو `Promise`.
- `inject`: اعتماديات المصنع بالترتيب.
- `useClass`: صنف يُنشأ ويطبّق `ConfigurableModuleOptionsFactory`، والدالة الافتراضية `create()`.
- `useExisting`: يعيد مزودًا موجودًا يطبّق الواجهة نفسها بدل نسخة جديدة.
- `provideInjectionTokensFrom`: مع المصنع، قائمة مزودات يلتقط منها الإطار ما ذُكر في `inject` لتسجيله داخل الوحدة الديناميكية. يفيد عندما تمرر وحدة خياراتها إلى وحدة متداخلة.

واحد فقط من `useFactory` و `useClass` و `useExisting`.

```ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { IdGeneratorModule } from '../id-generator/id-generator.module.js';
import { UsersService } from './users.service.js';

@Module({
  imports: [
    IdGeneratorModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        prefix: config.getOrThrow<string>('USER_ID_PREFIX'),
      }),
    }),
  ],
  providers: [UsersService],
})
export class UsersModule {}
```

تنفيذ `registerAsync` على الصنف يرجع `DynamicModule` يسجل مزود الخيارات من المصنع ثم `IdGenerator`، مثل `forRootAsync`، من غير `global`.

صنف خيارات بدل المصنع:

```ts
import { ConfigurableModuleOptionsFactory, Injectable } from '@nestjs/common';

@Injectable()
export class UserIdOptions implements ConfigurableModuleOptionsFactory<{ prefix: string }, 'create'> {
  create(): { prefix: string } {
    return { prefix: 'usr' };
  }
}
```

الاستدعاء `registerAsync({ useClass: UserIdOptions })`. `useExisting` يشير إلى المزود نفسه إن كان مسجّلًا عند المستورد.

## Mental Model

```text
UsersModule
  registerAsync -> ConfigService -> { prefix: 'usr' } -> IdGenerator هنا
OrdersModule
  registerAsync -> { prefix: 'ord' } -> IdGenerator هناك
```

الاسم `Async` يعني أن بناء الخيار قد ينتظر، لا أن كل طلب أصبح غير متزامن.

## أبسط مثال

مقطع `UsersModule` أعلاه. الطلبات تكرر النداء بمفتاح `ORDER_ID_PREFIX`. لا تضع النداءين في الجذر باسم `forRootAsync` إن كان المقصود خيارين متعايشين.

## شرح المثال

`imports: [ConfigModule]` يفتح `ConfigService` للمصنع. `getOrThrow` يفشل إقلاع هذه الوحدة إن غاب المفتاح، والأوراق تُهيأ قبل الجذر فالفشل يظهر قبل `listen`. الخدمة ما زالت تحقن رمز الخيارات الذي تسجله دالتك `registerAsync`.

## ماذا يحدث داخليًا؟

1. بناء `UsersModule` يرى كائن `registerAsync`.
2. تُستورد وحدات `imports`.
3. يُحل `inject` ويُستدعى المصنع ويُنتظر.
4. تُسجل المزودات في الوحدة الديناميكية الناتجة.
5. `UsersService` تحقن `IdGenerator` الخاص بهذا الاستيراد.
6. ميزة أخرى لها مصنعها. لا مشاركة خيار عبر `global`.

## مثال عملي

`useClass` عندما بناء السابقة فيه شرط أكثر من سطر: قراءة ملف مناطق أو اختيار حسب `NODE_ENV` داخل `create()`. الصنف قابل للاختبار وحده. `useExisting` عندما `AppModule` يملك `UserIdOptions` أصلًا وتريد الوحدة الديناميكية أن تستعمله لا أن تنشئ ثانيًا. `provideInjectionTokensFrom` عندما هذه الوحدة تبني وحدة متداخلة وتريد تمرير رموز من مزوداتها إلى `inject` الخاص بالنداء الداخلي من غير إعادة تسجيل زائدة.

## أخطاء شائعة

- جمع `useFactory` مع `useClass`.
- ترتيب `inject` لا يطابق المعاملات.
- `registerAsync` في الجذر لكل المفاتيح ثم افتراض عزل الميزات.
- نسيان أن `create` هو الاسم الافتراضي حتى يغيّره `ConfigurableModuleBuilder`.
- مصنع يرجع `prefix` بينما الخدمة تنتظر كائن `{ prefix }`.

## متى نستخدمه؟

خيار لكل مستورد يعتمد على الإعداد أو على وعد.

## متى لا نستخدمه؟

قيمة معروفة في سطر الاختبار: `register({ prefix: 'usr' })`. إعداد واحد للتطبيق: `forRootAsync`.

## مقارنة

| | `register` | `registerAsync` |
| --- | --- | --- |
| الخيار | كائن جاهز | مصنع أو `create` |
| `ConfigService` | تقرأه قبل النداء | داخل `inject` |
| العزل بين الميزات | نداء لكل وحدة | نداء لكل وحدة |
| الحقول المتنافية | — | `useFactory` / `useClass` / `useExisting` |

## تمارين

### تمرين سهل

ما الدالة الافتراضية التي يجب أن يوفّرها `useClass`؟

<details>
<summary>الحل</summary>

`create()`، وتعيد كائن الخيارات، إلا إذا غيّر البنّاء اسمها.

</details>

### تمرين متوسط

لماذا نداء المستخدم ونداء الطلب لا يُجمعان في `forRootAsync` واحد؟

<details>
<summary>الحل</summary>

لأن كل ميزة لها سابقة. `registerAsync` يبني خيارًا عند المستورد. `forRootAsync` تكوين مرة.

</details>

### تمرين متقدم

متى تختار `useExisting` بدل `useClass`؟

<details>
<summary>الحل</summary>

عندما المزود الذي يبني الخيارات مسجّل أصلًا وتريد النسخة نفسها، لا نسخة ثانية من الصنف.

</details>

## أسئلة مراجعة

- ما الحقول الثلاثة المتنافية؟
- ماذا يفعل `imports` داخل الكائن؟
- أين يعيش النداء: الجذر أم الميزة؟

## الخلاصة

`registerAsync` هو `register` عندما يُصنع الخيار بحقن وانتظار. نداء لكل مستورد، ومصنع واحد من ثلاثة أساليب.

## ماذا بعد؟

`ConfigurableModuleBuilder` يولّد `register` و `registerAsync` حتى لا تكتبهما يدويًا.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
