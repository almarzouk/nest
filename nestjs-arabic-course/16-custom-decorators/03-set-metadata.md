# SetMetadata

## الهدف من هذا الدرس

لصق بيانات ثابتة على الدالة أو الصنف بمفتاح، حتى يقرأها الحارس لاحقًا.

## المشكلة التي يحلها

الاسترداد يحتاج دور `refund` بينما القائمة مفتوحة لكل شباك معروف. الشرط إن كُتب داخل اسم الدالة أو داخل `if (url)` ينكسر مع البادئة. البيانات على الدالة تبقى مع الدالة حين يتغير المسار.

## الفكرة ببساطة

`SetMetadata(key, value)` من `@nestjs/common` تخزن `value` تحت `key` على الهدف الذي تزيّنه. الحارس لا يُكتب في هذا الدرس. هنا اللصق والمفتاح الثابت فقط. القراءة أداة `Reflector` في الدرس التالي.

## التعريف التقني

```ts
SetMetadata(metadataKey: string, metadataValue: unknown): CustomDecorator
```

المفتاح سلسلة تشاركها الزينة والقارئ. عرّفها مرة:

```ts
export const DESK_ROLES_KEY = 'deskRoles';
export const DeskRoles = (...roles: string[]) => SetMetadata(DESK_ROLES_KEY, roles);
```

`DeskRoles` دالة ترجع الزينة حتى لا يتكرر المفتاح في المتحكمات. هذا نمط الصفحة المنخفض قبل `Reflector.createDecorator`. الشكل الذي يفضّله CLI 12 يُعرض في درس `Reflector`. هنا ترى ما يخزنه المفتاح صراحة حتى تعرف ماذا يخفي الشكل الأحدث.

الاستخدام:

```ts
@SetMetadata(DESK_ROLES_KEY, ['refund'])
@Post('refund')
refund() {}
```

أو `@DeskRoles('refund')`.

القيمة تُخزن عند تحميل الصنف لا عند كل طلب. لا تضع فيها `request` ولا مقعدًا. تلك حالة طلب. البيانات هنا إعداد للدالة: أدوار، أو علم عام.

يمكن وضع الزينة على الصنف فتطبق على دواله حسب طريقة القراءة لاحقًا. `get` على الدالة لا يرى زينة الصنف تلقائيًا. `getAllAndOverride` يدمج المستويين. ذلك الدرس التالي. هنا اعرف أن مكان الزينة يغيّر أين تُخزن.

## Mental Model

```text
@DeskRoles('refund') على refund
  المفتاح deskRoles → ['refund']
الدالة list بلا زينة
  لا قيمة تحت المفتاح
```

## أبسط مثال

```ts
import { SetMetadata } from '@nestjs/common';

export const DESK_ROLES_KEY = 'deskRoles';

export const DeskRoles = (...roles: string[]) => SetMetadata(DESK_ROLES_KEY, roles);
```

```ts
import { DeskRoles } from './desk-roles.decorator.js';

@Controller('tickets')
export class TicketsController {
  @DeskRoles('refund')
  @Post('refund')
  refund(): { ok: true } {
    return { ok: true };
  }
}
```

`refund` تحمل `['refund']`. `list` إن أُضيفت بلا زينة لا تحمل المفتاح. الحارس الذي لم يُكتب بعد هو من سيمنع. في هذا الدرس يكفي أن البيانات لُصقت.

## شرح المثال

- `...roles` تجمع `@DeskRoles('refund', 'audit')` في مصفوفة واحدة تُخزن كقيمة.
- المفتاح ثابت مُصدَّر. الحارس يستورد `DESK_ROLES_KEY` لا السلسلة يدويًا.
- الملف ينتهي استيراده بـ `.js`.
- لا منطق «هل المستخدم مشرف» في `DeskRoles`. الدالة ترجع زينة فقط.

## ماذا يحدث داخليًا؟

عند تعريف الصنف تستدعي Nest الزينة وتكتب القيمة في خريطة بيانات الهدف عبر `Reflect.defineMetadata` تقريبًا. المفتاح هو `deskRoles`. عند الطلب لا تُعاد الكتابة. `Reflector.get` يقرأ الخريطة من مرجع `getHandler()` أو `getClass()`.

إن اختلفت السلسلة في الحارس عن `DESK_ROLES_KEY` فالقراءة `undefined` والحارس الذي يعتبر الغياب سماحًا يفتح الاسترداد. لذلك التصدير من ملف واحد هو الدرس العملي.

البيانات مشتركة بين كل الطلبات لأنها على الدالة لا على الطلب. تغيير المصفوفة بعد التحميل يغيّر كل الشباك. لا تعمل `push` على المصفوفة المخزنة أثناء الطلب.

## مثال عملي

مسار الاسترداد يُعلَّم `@DeskRoles('refund')` ومسار الإلغاء `@DeskRoles('cancel')`. مراجعة الملف ترى الدور بجانب الدالة من غير فتح الحارس. اختبار وحدة يستدعي `Reflect.getMetadata(DESK_ROLES_KEY, TicketsController.prototype.refund)` أو `Reflector` في الدرس التالي ويتوقع `['refund']`. لا HTTP لازم لإثبات اللصق.

## أخطاء شائعة

- مفتاح `'roles'` هنا و `'deskRoles'` في الحارس.
- تخزين المستخدم الحالي في `SetMetadata`. المستخدم يتغير كل طلب.
- توقع أن الزينة وحدها ترجع `403`.
- تعديل المصفوفة المرجعة أثناء الطلب.

## متى نستخدمه؟

لبيانات ثابتة على الدالة أو الصنف: أدوار، علم، اسم سياسة. غلاف `DeskRoles` يمنع تكرار المفتاح.

## متى لا نستخدمه؟

لقيمة تُستخرج من الطلب إلى معامل. ذلك `createParamDecorator`. وللشكل الذي يولّده CLI حديثًا إن بدأتم ملف زينة جديدًا بعد درس `Reflector`. المفتاح الصريح يبقى مفهومًا لما هو مخزون.

## مقارنة

| | `SetMetadata` | `createParamDecorator` |
| --- | --- | --- |
| تُوضع على | دالة أو صنف | معامل |
| الناتج | بيانات للقارئ | قيمة الوسيط |
| متى تُقرأ | في الحارس عادة | عند بناء الوسائط |
| الحالة | ثابتة على الهدف | من طلب هذه الدورة |

## تمارين

### تمرين سهل

ماذا يُخزن `@DeskRoles('refund')` تحت `deskRoles`؟

<details>
<summary>الحل</summary>

المصفوفة `['refund']`.

</details>

### تمرين متوسط

لماذا يُصدَّر `DESK_ROLES_KEY` بدل كتابة `'deskRoles'` في الحارس؟

<details>
<summary>الحل</summary>

حتى يتشارك اللصق والقراءة المرجع نفسه. اختلاف حرف في السلسلة يجعل `get` يرجع `undefined`.

</details>

### تمرين متقدم

تريد أن كل دوال المتحكم تحتاج `audit` إلا `refund` التي تحتاج `refund`. أين تلصق كل قيمة، وما الذي لا يضمنه هذا الدرس وحده؟

<details>
<summary>الحل</summary>

`@DeskRoles('audit')` على الصنف و `@DeskRoles('refund')` على الدالة. ظهور قيمة الدالة بدل الصنف يعتمد على قارئ `getAllAndOverride` في الدرس التالي. `SetMetadata` وحده يلصق في المكانين ولا يختار الفائز.

</details>

## أسئلة مراجعة

- ما وسيطا `SetMetadata`؟
- لماذا لا تُخزن بيانات الطلب؟
- هل الزينة تمنع الاسترداد وحدها؟
- ماذا يصدّر الملف غير الدالة؟

## الخلاصة

`SetMetadata` تلصق قيمة ثابتة بمفتاح على الدالة أو الصنف. غلاف صغير يحصر المفتاح. القارئ والحسم بين زينة الصنف والدالة درس `Reflector`، والشكل المفضّل للتوليد بعده مباشرة.

## ماذا بعد؟

الدرس التالي `Reflector`: القراءة، وتغليب الدالة على الصنف، و `createDecorator` الذي يفضّله CLI.

## المصادر الرسمية

- [Guards](https://docs.nestjs.com/guards)
- [Custom decorators](https://docs.nestjs.com/custom-decorators)
