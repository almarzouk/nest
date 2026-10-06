# @Public

## الهدف من هذا الدرس

تعليم مسار أو متحكم بأنه يتجاوز حارس الهوية العام، حتى تبقى الصحة وتسجيل الدخول مفتوحين.

## المشكلة التي يحلها

`APP_GUARD` على `IdentifyDeskGuard` يغلق كل الدوال. `GET /health` و `POST /auth/login` يرجعان `401` قبل أن يُقرأ الجسم. `exclude` في Middleware لا يصل إلى الحارس. العلامة على الدالة هي الاستثناء الذي يقرأه الحارس نفسه.

## الفكرة ببساطة

`@Public(true)` تلصق علمًا عبر `Reflector.createDecorator<boolean>()`. أول أسطر الحارس العام تقرأ العلم بـ `getAllAndOverride`. إن كان `true` ترجع `true` من غير ترويسة. غير ذلك تكمل فحص الهوية.

## التعريف التقني

شكل CLI للزينة المولَّدة، لا مفتاح نصي جديد في كل حارس:

```ts
import { Reflector } from '@nestjs/core';

export const Public = Reflector.createDecorator<boolean>();
```

الاستخدام `@Public(true)` على الدالة أو على صنف متحكم الدخول. القراءة:

```ts
const isPublic = this.reflector.getAllAndOverride(Public, [
  context.getHandler(),
  context.getClass(),
]);
if (isPublic) {
  return true;
}
```

الدالة قبل الصنف حتى يستطيع مسار واحد داخل متحكم محمي أن يُفتح، أو العكس إن علّمت الصنف ثم غطيت دالة لاحقًا. لتغليق دالة داخل صنف معلّم، `createDecorator` يخزن `false` أيضًا إن استدعيت `@Public(false)`، و `getAllAndOverride` يعتبر `false` قيمة موجودة فيغلب الإغلاق. لا تعتمد على حذف الزينة لإغلاق دالة على صنف مفتوح. ضع `@Public(false)` على الاستثناء.

الشكل الأقدم `() => SetMetadata(IS_PUBLIC_KEY, true)` يعطي `@Public()` بلا وسيط. يبقى مفهومًا من درس `SetMetadata`. الزينة الجديدة في هذا المشروع تتبع `createDecorator` وسيطًا منطقيًا كما يفعل CLI مع الزينة المولَّدة.

الحارس يجب أن يُحدَّث. العلامة وحدها لا تغيّر `401`.

`@Public(true)` لا يلغي `RolesGuard` إن كان عامًا أيضًا ويمنع عند الغياب. إن كانت الصحة بلا أدوار وحارس الأدوار يسمح عند الغياب، تكفي فتح الهوية. إن كان حارس الأدوار يرفض الغياب، علّمه أن يمرر `@Public` أيضًا أو لا تربطه على الصحة.

## Mental Model

```text
APP_GUARD IdentifyDeskGuard
  يقرأ @Public(true) → return true
  وإلا يفحص x-desk-name
GET /health معلّم
POST /tickets غير معلّم → 401 بلا ترويسة
```

## أبسط مثال

```ts
import { type CanActivate, type ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Public } from './public.decorator.js';

@Injectable()
export class IdentifyDeskGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride(Public, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    const request = context.switchToHttp().getRequest<{ headers: Record<string, string | undefined> }>();
    const name = request.headers['x-desk-name'];
    if (!name) {
      throw new UnauthorizedException();
    }
    request.user = { name, roles: ['audit'] };
    return true;
  }
}
```

```ts
@Public(true)
@Get('health')
health(): { ok: true } {
  return { ok: true };
}
```

`GET /health` يعمل بلا ترويسة. `POST /tickets` بلا ترويسة يبقى `401`.

## شرح المثال

- العلم يُفحص قبل الترويسة. هذا ترتيب الاستثناء.
- `request.user` لا يُلصق على الصحة. `@CurrentUser()` هناك تكون `undefined`. لا تضعها على `health`.
- `@Public(true)` على `AuthController` يفتح كل دوال الدخول. أدق من فتح التطبيق.
- لا `exclude('health')` في `configure`. ذلك ثقب Middleware لا ثقب حارس.

## ماذا يحدث داخليًا؟

الحارس العام يعمل على كل المسارات بعد Middleware. `getAllAndOverride` يقرأ بيانات الدالة ثم الصنف. `true` ينهي `canActivate` بنجاح فتكمل الدورة من غير `401`. الأنابيب على `login` تعمل بعد ذلك وتفحص كلمة المرور كشكل، ثم الخدمة تتحقق من السر. قبل العلامة كانت الدورة تموت في الحارس والجسم لا يُفحص.

`false` الصريح على دالة داخل صنف `@Public(true)` يُقرأ أولًا فيغلب. الصحة التي هي الدالة الوحيدة المعلمة لا تحتاج ذلك.

فلتر المسار لا علاقة له بالفتح. الفتح قرار بوابة لا شكل خطأ.

## مثال عملي

بعد نقل الهوية إلى `APP_GUARD` أُضيف `@Public(true)` على `health` وعلى `login` فقط. اختبار الصحة بلا ترويسة يتوقع `{ ok: true }`. اختبار `POST /tickets` بلا ترويسة يتوقع `401`. اختبار الدخول بجسم صالح شكليًا يصل إلى الخدمة. حذف سطر `if (isPublic)` يعيد إغلاق الصحة رغم بقاء الزينة، وهو اختبار يثبت أن القارئ جزء من العلامة.

## أخطاء شائعة

- الزينة من غير فرع في الحارس.
- `exclude` في Middleware بدل العلامة.
- `@Public(true)` على `TicketsController` فيفتح الاسترداد.
- لصق مستخدم وهمي داخل فرع العام. المسار العام بلا هوية.

## متى نستخدمه؟

على الصحة وتسجيل الدخول وما شابههما حين تكون الهوية `APP_GUARD`. وعلى دالة واحدة داخل متحكم مغلق إن كانت تلك الدالة عامة فعلًا.

## متى لا نستخدمه؟

لا تعلّم كل المتحكمات لتهدئة `401`. أعد الحارس إلى مستوى المتحكم إن كانت الميزات المحمية قليلة. لا تستخدم العلامة مكان `@Roles`.

## مقارنة

| الأداة | تفتح الصحة أمام حارس عام؟ |
| --- | --- |
| `exclude` في Middleware | لا |
| `@Public(true)` بلا قارئ | لا |
| `@Public(true)` وفرع أول في الحارس | نعم |
| إزالة `APP_GUARD` | نعم، وتغلق التذاكر أيضًا إن لم تُنقل الزينة |

## تمارين

### تمرين سهل

ماذا تضع فوق `health`، وما الوسيط الذي يمر إلى `createDecorator<boolean>`؟

<details>
<summary>الحل</summary>

`@Public(true)`. الوسيط `true`.

</details>

### تمرين متوسط

لماذا يبقى `401` بعد إضافة الزينة إن لم يتغير الحارس؟

<details>
<summary>الحل</summary>

العلامة بيانات على الدالة. `canActivate` ما زال يفحص الترويسة أولًا. أضف قراءة `Public` وارجع `true` قبل الفحص.

</details>

### تمرين متقدم

متحكم الدخول كله `@Public(true)` وفيه دالة `logout` يجب أن تبقى بهوية. كيف تغلق `logout` فقط؟

<details>
<summary>الحل</summary>

`@Public(false)` على `logout`، والقارئ `getAllAndOverride` يفحص الدالة قبل الصنف. `false` قيمة موجودة فتغلب علم الصنف. حذف الزينة عن `logout` يترك علم الصنف `true` ويبقيها مفتوحة.

</details>

## أسئلة مراجعة

- لماذا لا يكفي `exclude`؟
- أين يُفحص العلم داخل `canActivate`؟
- ماذا يصل إلى `@CurrentUser` على مسار عام؟
- كيف تُغلق دالة واحدة داخل صنف معلَّم؟

## الخلاصة

`@Public(true)` استثناء يقرأه الحارس العام قبل فحص الهوية. الصحة والدخول يُعلَّمان، والتذاكر لا تُعلَّم. `false` على الدالة يغلب علم الصنف. الزينة بلا فرع في الحارس لا تفتح شيئًا.

## ماذا بعد؟

أمثلة الفصل تجمع المستخدم والأدوار والعلم العام، ثم التمارين، ثم فصل الإعداد حيث تختلف القيم بين البيئات لا بين الدوال.

## المصادر الرسمية

- [Custom decorators](https://docs.nestjs.com/custom-decorators)
- [Guards](https://docs.nestjs.com/guards)
- [Execution context](https://docs.nestjs.com/fundamentals/execution-context)
