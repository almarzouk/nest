# @Roles

## الهدف من هذا الدرس

تعليم الدالة بالأدوار المطلوبة ومنع من لا يحملها، عبر زينة `Reflector.createDecorator` وحارس يقرأها.

## المشكلة التي يحلها

`refund` للدور `refund` و `cancel` للدور `cancel`. شرط `if (user.name === 'lead')` داخل الخدمة يخفي السياسة عن المراجعة ويكسر حين يتغير اسم الشخص لا دوره.

## الفكرة ببساطة

`@Roles(['refund'])` تلصق المصفوفة بالدالة بالشكل الذي يولّده CLI. الحارس يحقن `Reflector`، يقرأ الزينة، ويقارنها بـ `request.user.roles`. غياب الزينة في هذا التصميم يعني أن المسار لا يطلب دورًا إضافيًا فيرجع `true` بعد أن تكون الهوية قد لُصقت في حارس أسبق. وجود الزينة مع نقص الدور يرجع `false` فيصبح `403`.

## التعريف التقني

```ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

```ts
@Roles(['refund'])
@Post('refund')
refund() {}
```

القراءة كما في صفحة الحراس:

```ts
const roles = this.reflector.getAllAndOverride(Roles, [
  context.getHandler(),
  context.getClass(),
]);
if (!roles) {
  return true;
}
```

ثم مطابقة مع `user.roles`. الصفحة تترك `matchRoles` لكم. هنا المطابقة: يكفي أن يحمل المستخدم أحد الأدوار المطلوبة.

الحارس يُربط بعد حارس الهوية في `@UseGuards(IdentifyDeskGuard, RolesGuard)` أو بتسجيل `APP_GUARD` للهوية أولًا. `false` يعني `403` لا `401`. الهوية الناقصة تُعالج في الحارس الأول.

لا تضع `@Roles` على معامل. هي زينة دالة أو صنف.

## Mental Model

```text
هوية تلصق user.roles
@Roles(['refund']) على الدالة
RolesGuard: تقاطع غير فارغ → true
            لا تقاطع → false → 403
            لا زينة → true
```

## أبسط مثال

```ts
import { type CanActivate, type ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator.js';

type DeskRequest = { user?: { name: string; roles: string[] } };

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride(Roles, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!required || required.length === 0) {
      return true;
    }
    const request = context.switchToHttp().getRequest<DeskRequest>();
    const owned = request.user?.roles ?? [];
    return required.some((role) => owned.includes(role));
  }
}
```

```ts
@Controller('tickets')
@UseGuards(IdentifyDeskGuard, RolesGuard)
export class TicketsController {
  @Roles(['refund'])
  @Post('refund')
  refund(@CurrentUser('name') name: string): { name: string } {
    return { name };
  }
}
```

`IdentifyDeskGuard` يجب أن يلصق `roles` لا الاسم فقط حتى تنجح المطابقة. إن كان اللصق القديم `{ name }` فقط فالمصفوفة فارغة والاسترداد `403` دائمًا.

## شرح المثال

- `some` تعني «أحد الأدوار يكفي». إن كانت السياسة «كل الأدوار» فالاستبدال `every`.
- `getAllAndOverride` يجعل `@Roles(['cancel'])` على الدالة تغلب `@Roles(['audit'])` على الصنف.
- `@CurrentUser` في الدالة يعمل بعد `true` فقط.
- الزينة `['refund']` مصفوفة كما في مثال التوثيق، لا `@Roles('refund')` إلا إذا غيّرت نوع `createDecorator` إلى معمل بقية. التوثيق الحالي وسيط واحد هو `string[]`.

## ماذا يحدث داخليًا؟

عند التحميل تُخزن `['refund']` على `refund`. عند الطلب يعمل حارس الهوية ثم `RolesGuard`. القراءة لا تستدعي الدالة. `false` يرمي `ForbiddenException` قبل الأنبوب. `true` يكمل وقد يصل إلى `@CurrentUser`.

غياب `@Roles` يرجع `true` حتى لا تُغلق كل دوال المتحكم إن نسيت الزينة. إن أردت الإغلاق الافتراضي فاعكس الشرط: الغياب `false`، وعلّم الدوال العامة بـ `@Public` في الدرس التالي لا بغياب الأدوار. لا تخلط العلمين في شرط واحد غامض.

حارس عام `RolesGuard` بلا هوية قبله يرى `roles` فارغة فيرفض كل مسار معلّم ويمرر غير المعلم. الترتيب جزء من صحة `@Roles`.

## مثال عملي

شباك `lead` يحمل `roles: ['refund', 'audit']` فيسترد. شباك `guest` يحمل `['audit']` فيأخذ `403` على `refund` ويمضي على `GET` بلا `@Roles`. اختبار الحارس بسياق `refund` ومستخدم `guest` يتوقع `false`. اختبار بلا زينة على `list` يتوقع `true`. لا تختبر اسم الشخص.

## أخطاء شائعة

- `@Roles('refund')` بينما النوع `string[]` فيصبح المخزون سلسلة والمطابقة `some` على المحارف.
- مقارنة `user.name` داخل `RolesGuard`.
- تسجيل الحارس قبل حارس اللصق.
- اعتبار الغياب منعًا ثم الدهشة أن `list` مفتوحة. هذا الدرس يفتح غير المعلم عمدًا.

## متى نستخدمه؟

حين تختلف الدوال في الدور المطلوب، والدور قائمة على المستخدم لا اسم شخص.

## متى لا نستخدمه؟

لمسار بلا تمييز دور. لا تضع `@Roles(['audit'])` على الصحة. ولإثبات الهوية. ذلك الحارس السابق ورمزه `401`.

## مقارنة

| الحالة | الناتج |
| --- | --- |
| لا `@Roles` | `true` في هذا الحارس |
| المستخدم يحمل دورًا مطلوبًا | `true` |
| لا يحمل أيًا منها | `false` ثم `403` |
| لا `user` لأن الهوية لم تلصق | `false` على المسار المعلم |

## تمارين

### تمرين سهل

ماذا تضع فوق `refund` حتى يُطلب دور الاسترداد بالشكل الموثّق؟

<details>
<summary>الحل</summary>

`@Roles(['refund'])` والثابت من `Reflector.createDecorator<string[]>()`.

</details>

### تمرين متوسط

لماذا `GET` بلا زينة يمر من `RolesGuard` بينما `guest` يُمنع من `refund`؟

<details>
<summary>الحل</summary>

غياب الزينة يرجع `true`. `refund` تحمل `['refund']` و `guest` لا يملكها فالناتج `false`.

</details>

### تمرين متقدم

الصنف `@Roles(['audit'])` والدالة `@Roles(['refund'])`. أي مصفوفة تُقارَن، وما ترتيب أهداف `getAllAndOverride`؟

<details>
<summary>الحل</summary>

`['refund']` لأن الدالة تُفحص قبل الصنف: `[context.getHandler(), context.getClass()]`. عكس المصفوفة يقارن `audit` ويخطئ الاستثناء.

</details>

## أسئلة مراجعة

- ما نوع وسيط `@Roles` في هذا الفصل؟
- متى يرجع الحارس `true` بلا فحص مستخدم؟
- لماذا الترتيب بعد حارس الهوية؟
- ماذا يرى العميل عند نقص الدور؟

## الخلاصة

`@Roles(['...'])` سياسة على الدالة يقرأها `RolesGuard` من `Reflector` ويطابقها مع `user.roles`. الغياب هنا سماح إضافي لا منع. النقص `403` بعد أن تكون الهوية معروفة.

## ماذا بعد؟

الدرس التالي `@Public`: علامة تفتح مسارًا أمام حارس عام يغلق الصحة وتسجيل الدخول.

## المصادر الرسمية

- [Guards](https://docs.nestjs.com/guards)
- [Custom decorators](https://docs.nestjs.com/custom-decorators)
