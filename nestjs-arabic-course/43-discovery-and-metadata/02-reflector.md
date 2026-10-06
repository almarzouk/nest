# Reflector

## الهدف من هذا الدرس

قراءة metadata التي لصقها `SetMetadata` عبر `Reflector`، مع تغليب الدالة على الصنف أو دمج المصفوفات.

## المشكلة التي يحلها

`Reflect.getMetadata` مباشرة يكرر المفاتيح وقواعد «الدالة تغلب الصنف». الحارس الذي ينسخ القاعدة يخطئ في مسار عام على الصنف ومسار استثناء على الدالة. `Reflector` من `@nestjs/core` يجمع الاستعلام: مفتاح، وهدف أو قائمة أهداف.

## الفكرة ببساطة

احقن `Reflector` في Guard أو Interceptor. `get` يسأل هدفًا واحدًا. `getAllAndOverride` يمشي قائمة أهداف ويعيد أول قيمة معرّفة، فتضع الدالة أولًا ثم الصنف ليغلب المسار على المتحكم. `getAllAndMerge` يجمع المصفوفات من الأهداف. `Reflector.createDecorator` ينشئ مزخرفًا ورمز قراءة معًا حتى لا تكتب سلسلة المفتاح بيدك.

## التعريف التقني

```ts
import { Reflector } from '@nestjs/core';

constructor(private readonly reflector: Reflector) {}
```

قراءة علم عام كما في توثيق الحراس:

```ts
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),
  context.getClass(),
]);
```

`getHandler()` الدالة و `getClass()` الصنف، من `ExecutionContext` في الدرس الأخير من الفصل. ترتيب المصفوفة هو ترتيب التغليب: الأول المعرّف يفوز. لذلك الدالة قبل الصنف تعني أن `@Public()` على المسار يسمح حتى لو كان الصنف بلا العلم، وأن علمًا على الصنف يسري على الدوال التي لم تعرّف المفتاح.

`getAllAndMerge` عندما تجمع أدوار الصنف مع أدوار الدالة في قائمة واحدة بدل إسقاط أحدهما.

`get(key, target)` عندما تعرف أن المفتاح على الدالة فقط.

مزخرف مولَّد:

```ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

الاستعمال `@Roles(['admin'])`، والقراءة `this.reflector.get(Roles, context.getHandler())`. النوع على `createDecorator` يوثق القيمة. `DiscoveryService.createDecorator` مسار آخر للتصفية في الدرس الخاص به، لا تستبدله هنا من غير حاجة للاكتشاف.

`Reflector` نفسه Singleton تحقنه. لا تنشئ `new Reflector()` في كل طلب إلا في اختبار يعزل القراءة.

## Mental Model

```text
المفتاح على الدالة؟ خذه
وإلا على الصنف؟ خذه
وإلا الافتراض في الحارس (غالبًا ممنوع)
```

هذا `getAllAndOverride` بالترتيب `[handler, class]`.

## أبسط مثال

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from './public.decorator.js';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    const request = context.switchToHttp().getRequest<{ user?: unknown }>();
    return Boolean(request.user);
  }
}
```

## شرح المثال

المسار `@Public()` يعيد `true` من الدالة فينتهي الفحص. مسار بلا المزخرف يسقط إلى الصنف ثم إلى `undefined`، فيُشترط `user`. الحارس لا يستورد قائمة المسارات العامة. القائمة هي المزخرفات نفسها.

## ماذا يحدث داخليًا؟

1. الإطار يبني الحارس كمزود ويحقن `Reflector`.
2. عند الطلب يمرر `ExecutionContext`.
3. `getAllAndOverride` يقرأ metadata الدالة ثم الصنف.
4. أول قيمة ليست `undefined` تُعاد.
5. `false` قيمة معرّفة وتغلب ما على الصنف. لا تخلط «غير موجود» مع `false` إن كان للاثنين معنى مختلف. للعلم العام استعمل `true` أو الغياب.
6. القراءة رخيصة نسبة إلى قاعدة البيانات. لا تغني عن التحقق من الرمز الموقّع. العلم يقول «لا تشترط جلسة»، ولا يمنح صلاحية.

## مثال عملي

أدوار تُدمج:

```ts
const roles = this.reflector.getAllAndMerge<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

صنف `@Roles('staff')` ودالة `@Roles('payroll')` ينتجان قائمة فيها الاثنان إن كان المخزن مصفوفات. غيّر إلى `getAllAndOverride` إن كانت الدالة يجب أن تستبدل أدوار الصنف لا أن تضيف إليها. الاتفاق يُكتب في الحارس ويُذكر بجانب المزخرف.

## أخطاء شائعة

- عكس ترتيب `[class, handler]` ثم استغراب أن الدالة لا تغلب.
- مفتاح المزخرف مختلف عن مفتاح `get`.
- `new Reflector()` داخل الدالة فيتجاوز أي بديل اختبار.
- اعتبار القراءة تفويضًا. ما زلت تتحقق من الفاعل على الطلب.
- `getAllAndMerge` على قيمة ليست مصفوفة.

## متى نستخدمه؟

حارس أو معترض يقرأ لصقًا على المسار أو على المتحكم.

## متى لا نستخدمه؟

لقراءة جسم الطلب أو الترويسة. تلك `switchToHttp().getRequest()`. ولا لتسجيل مزودات جديدة. ذلك `providers` أو `DiscoveryService` إن كنت تمسح الرسم.

## مقارنة

| الدالة | الأهداف | الناتج |
| --- | --- | --- |
| `get` | واحد | قيمة ذلك الهدف أو فراغ |
| `getAllAndOverride` | بالترتيب | أول قيمة معرّفة |
| `getAllAndMerge` | بالترتيب | دمج المصفوفات |

## تمارين

### تمرين سهل

من أين تُحقن `Reflector`؟

<details>
<summary>الحل</summary>

من `@nestjs/core`، كأي مزود في الـ `constructor`.

</details>

### تمرين متوسط

`@Public()` على الدالة والصنف بلا علم. ماذا يعيد `getAllAndOverride` إن كانت الدالة أول عنصر؟

<details>
<summary>الحل</summary>

`true` من الدالة، ولا يكمل إلى الصنف.

</details>

### تمرين متقدم

متى تختار `getAllAndMerge` لأدوار ومتى `getAllAndOverride`؟

<details>
<summary>الحل</summary>

الدمج إن كانت أدوار الدالة تُضاف إلى أدوار الصنف. التغليب إن كانت الدالة تستبدل قائمة الصنف بالكامل.

</details>

## أسئلة مراجعة

- ماذا يعني ترتيب مصفوفة الأهداف؟
- ماذا يوفّر `createDecorator` زيادة على `SetMetadata`؟
- لماذا `false` يختلف عن غياب المفتاح؟

## الخلاصة

`Reflector` يقرأ مفتاحًا من الدالة والصنف. غلّب بـ `getAllAndOverride` وادمج بـ `getAllAndMerge`، وثبّت المفتاح مع المزخرف.

## ماذا بعد؟

Metadata: ماذا تلصق، وعلى أي هدف، وكيف تسمي المفتاح.

## المصادر الرسمية

- NestJS — Custom decorators: https://docs.nestjs.com/custom-decorators
- NestJS — Guards: https://docs.nestjs.com/guards
