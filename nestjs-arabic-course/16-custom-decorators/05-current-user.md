# @CurrentUser

## الهدف من هذا الدرس

تمرير المستخدم الذي لُصق على الطلب إلى معامل الدالة، مع اختيار حقل واحد عند الحاجة.

## المشكلة التي يحلها

`refund` و `list` و `audit` تكرر `request.user` وقد تنسى أن الحقل قد يكون غائبًا. التوقيع لا يقول إن العملية تعمل باسم شباك. زينة المعامل تجعل الوسيط هو المستخدم لا الطلب.

## الفكرة ببساطة

`@CurrentUser()` ترجع كائن المستخدم. `@CurrentUser('name')` ترجع الحقل `name`. المصنع `createParamDecorator` يقرأ `request.user` الذي وضعه حارس الهوية قبل بناء الوسائط. الزينة لا تُسجّل أحدًا.

## التعريف التقني

الصفحة تبني `@User()` هكذا، ونسميها `@CurrentUser` في الشباك:

```ts
export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest<{ user?: DeskUser }>();
    const user = request.user;
    return data ? user?.[data as keyof DeskUser] : user;
  },
);
```

`data` هو النص داخل القوس. من غيره ترجع الكائن. الصفحة تستخدم `user?.[data]` حتى لا يرمي الوصول إن سبق الحارس ورفض، أو إن استُدعيت الزينة على مسار بلا لصق.

نوع `DeskUser` في هذا الدرس:

```ts
export type DeskUser = { name: string; roles: string[] };
```

الحارس `IdentifyDeskGuard` من فصل الحراس يضع `{ name }` على الأقل. توسيع اللصق ليشمل `roles` يتم هناك أو في حارس لاحق، لا داخل الزينة.

`ValidationPipe` لا يفحص هذا المعامل إلا مع `validateCustomDecorators`. لا نضع DTO المستخدم على الزينة. المستخدم ناتج بوابتكم لا جسم العميل.

## Mental Model

```text
IdentifyDeskGuard يلصق request.user
بعدها تُبنى الوسائط
@CurrentUser()      → الكائن
@CurrentUser('name') → النص
```

## أبسط مثال

```ts
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';

export type DeskUser = { name: string; roles: string[] };

export const CurrentUser = createParamDecorator(
  (data: keyof DeskUser | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest<{ user?: DeskUser }>();
    const user = request.user;
    if (!data) {
      return user;
    }
    return user?.[data];
  },
);
```

```ts
@Post('refund')
refund(@CurrentUser() user: DeskUser): { by: string } {
  return { by: user.name };
}
```

```ts
@Get('mine')
mine(@CurrentUser('name') name: string): { name: string } {
  return { name };
}
```

## شرح المثال

- `keyof DeskUser` يقيّد `'name'` و `'roles'` في المترجم.
- `user?.[data]` يعيد `undefined` إن لم يُلصق شيء، بدل رمي داخل الزينة.
- `refund` تطلب الكائن لأن العملية قد تسجّل الاسم والأدوار معًا.
- `mine` تطلب الاسم فقط حتى لا تصل مصفوفة الأدوار إلى دالة لا تحتاجها.
- الدالة لا تستدعي `switchToHttp`.

## ماذا يحدث داخليًا؟

الحارس يعمل أولًا. إن رمى `401` لا تُبنى الوسائط ولا تُستدعى الزينة. إن رجع `true` بعد اللصق، Nest تستدعي المصنع عند تجهيز معامل `refund`. القيمة تذهب إلى الدالة كما يخرج `@Body` بعد الأنبوب. لا أنبوب في المثال، فالقيمة هي مرجع الكائن على الطلب. لا تنسخ الزينة الكائن. تعديل `user.name` داخل الدالة يعدّل كائن الطلب نفسه. إن كان ذلك غير مرغوب فانسخ في الخدمة الحقول التي ستخزنها.

معترض بعد الحارس وقبل الأنبوب لا يزيل `request.user`. المعامل يُبنى قرب الدالة بعد الأنابيب الخاصة به. اللصق ما زال موجودًا.

اختبار المصنع: ابنِ سياقًا مزيفًا `switchToHttp().getRequest()` يرجع `{ user: { name: 'lead', roles: ['refund'] } }` واستدعِ منطق القراءة. `@CurrentUser('name')` يعطي `lead`.

## مثال عملي

سجل الاسترداد كان يكتب `request.user.name` وينكسر حين يكون الحقل `fullName` في خدمة أخرى. `@CurrentUser('name')` يحصر الاسم في نوع `DeskUser`. تغيير الحقل إلى `fullName` يفشل `tsc` في الزينة وفي الدوال التي استخدمت `'name'`. هذا أقوى من بحث نصي عن `request.user`. مسار الصحة بلا حارس لصق لا يستخدم الزينة. إن وُضعت سهوًا فالمعامل `undefined` ويجب ألا تُستدعى الخدمة.

## أخطاء شائعة

- بناء المستخدم من الترويسة داخل الزينة بينما الحارس يلصق شكلًا مختلفًا. مصدر واحد للّصق.
- افتراض أن `user` دائمًا معرّف لأن النوع في التوقيع ليس اختياريًا. النوع كذبة إن غاب اللصق. الحارس يضمن ذلك على المسارات المحمية، لا المترجم.
- تخزين `user` في متغير وحدة أثناء أول طلب.
- فحص دور الاسترداد داخل `@CurrentUser`. الدور لحارس `@Roles`.

## متى نستخدمه؟

على كل دالة محمية تحتاج اسم الشباك أو كائن المستخدم في الجسم أو في السجل، بعد حارس يلصق `request.user`.

## متى لا نستخدمه؟

على الصحة والمسار العام حيث لا لصق. وعلى قرار المنع. الزينة لا ترجع `false`. إن غاب المستخدم على مسار محمي فأصلح الحارس لا الزينة.

## مقارنة

| الاستخدام | الوسيط |
| --- | --- |
| `@CurrentUser()` | `DeskUser` أو `undefined` |
| `@CurrentUser('name')` | `string` أو `undefined` |
| `@Req()` | الطلب كله |
| `@Body()` | JSON العميل لا المستخدم |

## تمارين

### تمرين سهل

ماذا ترجع `@CurrentUser('roles')` إن كان المستخدم `{ name: 'lead', roles: ['refund'] }`؟

<details>
<summary>الحل</summary>

المصفوفة `['refund']`.

</details>

### تمرين متوسط

لماذا لا تُستدعى الزينة حين يرمي حارس الهوية `UnauthorizedException`؟

<details>
<summary>الحل</summary>

بناء الوسائط بعد نجاح الحراس. الرمي يوقف الدورة قبل المعامل.

</details>

### تمرين متقدم

الدالة تعدّل `user.roles.push('audit')` ثم طلب لاحق على الاتصال نفسه يرى الدور الزائد. ما سبب المشاركة، وما العادة الأنسب؟

<details>
<summary>الحل</summary>

الزينة أعادت مرجع كائن الطلب. التعديل يغيّر الحالة المشتركة لبقية الدورة. انسخ الحقول التي ستُخزن في سجل الاسترداد ولا تعدّل `roles` على الكائن الأصلي.

</details>

## أسئلة مراجعة

- من يلصق `request.user`؟
- ماذا يفعل الوسيط `data`؟
- هل الزينة ترمي `401`؟
- لماذا النوع في التوقيع لا يكفي إن غاب الحارس؟

## الخلاصة

`@CurrentUser` تقرأ ما لصقه الحارس. بلا وسيط ترجع الكائن، ومع اسم حقل ترجع ذلك الحقل. المنع والفحص ليسا عملها. لا تعدّل المرجع إن كانت بقية الطلب تشاركه.

## ماذا بعد؟

الدرس التالي `@Roles`: زينة الأدوار على الدالة وحارس يطابقها مع `user.roles`.

## المصادر الرسمية

- [Custom decorators](https://docs.nestjs.com/custom-decorators)
- [Guards](https://docs.nestjs.com/guards)
