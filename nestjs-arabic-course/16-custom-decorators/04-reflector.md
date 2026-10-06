# Reflector

## الهدف من هذا الدرس

قراءة البيانات التي لُصقت على الدالة أو الصنف، وبناء الزينة بـ `Reflector.createDecorator` كما يفضّل Nest CLI 12.

## المشكلة التي يحلها

`SetMetadata('deskRoles', ...)` يحتاج مفتاحًا نصيًا في الكاتب وفي القارئ. `Reflector.createDecorator` ينشئ الزينة والمرجع معًا فلا يُعاد كتابة السلسلة. ومن غير `getAllAndOverride` ترى الدالة زينتها فقط وتتجاهل زينة الصنف، أو العكس فيضيع استثناء الاسترداد.

## الفكرة ببساطة

`Reflector` من `@nestjs/core` يُحقن في الحارس. `get` يقرأ هدفًا واحدًا. `getAllAndOverride` يمشي قائمة أهداف وأول قيمة معرّفة تفوز، فتُمرَّر الدالة ثم الصنف حتى تغلب زينة المسار. `createDecorator<T>()` يصدر زينة تستقبل `T` وتُمرَّر هي نفسها إلى `get` بدل المفتاح النصي.

## التعريف التقني

الصفحة في الحراس تبني الأدوار هكذا:

```ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

الاستخدام `@Roles(['refund'])`. القراءة:

```ts
const roles = this.reflector.get(Roles, context.getHandler());
```

إن غابت الزينة عن الدالة فالناتج `undefined`.

لتغليب الدالة على الصنف:

```ts
this.reflector.getAllAndOverride(Roles, [
  context.getHandler(),
  context.getClass(),
]);
```

أول عنصر في المصفوفة يُفحص أولًا. ضع الدالة قبل الصنف. `getAllAndMerge` يدمج القيم حين يكون الدمج مطلوبًا لا الاستبدال. للأدوار التي تستبدل سياسة الصنف، `getAllAndOverride` هو المناسب.

`SetMetadata` ما زال يعمل. القارئ عندئذ `this.reflector.get(DESK_ROLES_KEY, handler)` بالمفتاح. الزينة الجديدة في المشروع تُولد بـ `createDecorator` حتى لا يوجد مفتاحان. لا تخلط الأسلوبين على السياسة نفسها.

`Reflector` مزوّد جاهز. لا تسجّله في `providers` عادة. احقنه في الباني.

## Mental Model

```text
createDecorator → Roles
@Roles(['refund']) على الدالة
@Roles(['audit']) على الصنف
getAllAndOverride(Roles, [handler, class]) → ['refund']
```

## أبسط مثال

```ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

```ts
import { Injectable, type CanActivate, type ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator.js';

@Injectable()
export class RolesReaderGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride(Roles, [
      context.getHandler(),
      context.getClass(),
    ]);
    return Array.isArray(roles);
  }
}
```

الحارس هنا يثبت القراءة فقط: `true` إن وُجدت مصفوفة أدوار على الدالة أو الصنف. مطابقة المستخدم بالدور درس `@Roles`. لا تفتح كل المسارات بهذا الشرط في الإنتاج. الغياب `undefined` يرجع `false` هنا، وهو اختيار تعليمي ضيق.

## شرح المثال

- `Roles` مرجع زينة لا سلسلة.
- ترتيب `[handler, class]` يجعل `@Roles` على `refund` تغلب `@Roles` على الصنف.
- الحقن `constructor` يعمل حين يُربَط الحارس كصنف لا `new RolesReaderGuard()`.
- `get` على `getHandler()` وحده يكفي حين لا تلصق شيئًا على الصنف.

## ماذا يحدث داخليًا؟

`createDecorator` يسجل مفتاحًا داخليًا خاصًا بالدالة التي أرجعها. عند `@Roles(['refund'])` تُخزن المصفوفة على الدالة. `getAllAndOverride` يقرأ بيانات الدالة فإن وجد قيمة غير `undefined` و غير `null` يعيدها، وإلا يقرأ الصنف. لا يستدعي `refund`.

القراءة في الحارس قبل الأنبوب والدالة. المعالج لا يستلم الأدوار إلا إذا قرأتها زينة معامل أيضًا. ابقِ السياسة في الحارس.

`Reflect.getMetadata` مباشرة يعمل مع `SetMetadata` إن عرفت المفتاح. مع `createDecorator` استخدم `Reflector` حتى لا تعتمد على سلسلة داخلية غير موثقة.

## مثال عملي

`TicketsController` عليه `@Roles(['audit'])` و `refund` عليها `@Roles(['refund'])`. اختبار الحارس يبني سياقًا فيه `getHandler` يشير إلى `refund` و `getClass` إلى الصنف، ويتوقع أن المصفوفة المقروءة `['refund']` لا `['audit']`. عكس ترتيب المصفوفة في `getAllAndOverride` يجعل الاختبار يتوقع `audit` وهذا خطأ مراجعة شائع. الدالة أولًا.

## أخطاء شائعة

- تمرير المفتاح النصي القديم إلى `get(Roles)` بعد أن صارت `Roles` ناتج `createDecorator`. استخدم المرجع نفسه.
- `get` على الصنف فقط فتضيع زينة الدالة.
- وضع الصنف قبل الدالة في `getAllAndOverride` فيغلب `audit` على `refund`.
- `new Reflector()` في كل طلب بدل الحقن. الحقن هو مسار الإطار.

## متى نستخدمه؟

عند كل زينة بيانات جديدة: `createDecorator` للكتابة، و `getAllAndOverride` إن كان للصنف والدالة معًا، و `get` على الدالة إن كانت الزينة على المسار فقط.

## متى لا نستخدمه؟

لا تقرأ `Reflector` من Middleware. السياق هناك لا يعطي `getHandler` كما في الحارس. ولا تستخدم `getAllAndMerge` للأدوار إن كنت تقصد الاستبدال لا اتحاد المجموعتين.

## مقارنة

| الأسلوب | الكتابة | القراءة |
| --- | --- | --- |
| مفتاح نصي | `SetMetadata(KEY, value)` | `reflector.get(KEY, target)` |
| CLI 12 | `Reflector.createDecorator<T>()` | `reflector.get(Decorator, target)` |
| صنف ودالة | الزينة على الاثنين | `getAllAndOverride(Decorator, [handler, class])` |

## تمارين

### تمرين سهل

من أي حزمة يُستورد `Reflector`؟

<details>
<summary>الحل</summary>

من `@nestjs/core`.

</details>

### تمرين متوسط

دالة بلا `@Roles` وصنف عليه `@Roles(['audit'])`. ماذا يعيد `getAllAndOverride` إن كانت الدالة أول عنصر؟

<details>
<summary>الحل</summary>

`['audit']`، لأن الدالة لا تحمل قيمة فيكمل القارئ إلى الصنف.

</details>

### تمرين متقدم

لماذا لا يمرَّر `'deskRoles'` إلى `get` بعد التحويل إلى `createDecorator`؟

<details>
<summary>الحل</summary>

المفتاح الداخلي تولّد مع الدالة `Roles`. القراءة بالمرجع `Roles`. السلسلة القديمة تشير إلى خريطة `SetMetadata` إن بقيت، لا إلى زينة `createDecorator`.

</details>

## أسئلة مراجعة

- ماذا يصدر `createDecorator<string[]>()`؟
- أي هدف يُفحص أولًا في `getAllAndOverride` كما نكتبه؟
- أين يُحقن `Reflector`؟
- متى يكفي `get` على `getHandler()`؟

## الخلاصة

`Reflector` يقرأ ما لصقته الزينة. `createDecorator` يلغي المفتاح النصي المتكرر وهو شكل CLI 12. `getAllAndOverride` مع الدالة قبل الصنف يجعل استثناء المسار يغلب سياسة المتحكم.

## ماذا بعد؟

الدرس التالي `@CurrentUser`: زينة معامل تقرأ المستخدم الذي لصقه الحارس، مع `data` لاختيار حقل.

## المصادر الرسمية

- [Guards](https://docs.nestjs.com/guards)
- [Custom decorators](https://docs.nestjs.com/custom-decorators)
