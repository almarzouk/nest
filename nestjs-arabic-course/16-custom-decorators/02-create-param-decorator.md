# createParamDecorator

## الهدف من هذا الدرس

بناء زينة معامل بدالة مصنع تستخرج قيمة من `ExecutionContext`.

## المشكلة التي يحلها

`@Req()` يمرر الطلب كله. استخراج ترويسة المكتب أو اسم الشباك يتكرر مع فحص أن الطلب HTTP. المصنع يضع الاستخراج في ملف واحد والوسيط في الدالة يصبح القيمة فقط.

## الفكرة ببساطة

`createParamDecorator((data, ctx) => value)` من `@nestjs/common`. `data` ما بين القوسين عند الاستخدام، مثل `'name'`. `ctx` سياق التنفيذ. القيمة الراجعة تصل إلى المعامل بعد الحارس، مثل `@Body()`.

## التعريف التقني

```ts
export const DeskName = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest<{ headers: Record<string, string | undefined> }>();
    return request.headers['x-desk-name'];
  },
);
```

الاستخدام `@DeskName() name: string`. الصفحة تقول إن `createParamDecorator` عام، فتستطيع تثبيت نوع `data`: `createParamDecorator((data: string, ctx) => ...)`.

`data` هنا غير مستخدم. الدرس عن `@CurrentUser` سيمرر اسم حقل. هذا الدرس يثبت المصنع نفسه على ترويسة واحدة.

الأنابيب تعمل على زينة المعامل المخصصة كما على `@Body`، لكن `ValidationPipe` يتخطاها ما لم تضع `validateCustomDecorators: true`. `StandardSchemaValidationPipe` كذلك يحتاج `validateCustomDecorators` وخيار `{ schema }` على الزينة، كما في صفحة التحقق. لا تفتح ذلك هنا. الاستخراج فقط.

لا تلصق `request.user` داخل المصنع كأثر جانبي على كل قراءة. اللصق للحارس. المصنع يقرأ.

المصنع يعمل عند بناء الوسائط لكل طلب. لا تغلق على طلب سابق في متغير في الملف.

## Mental Model

```text
@DeskName() name
    ↓ مصنع
ctx.switchToHttp().getRequest()
    ↓
headers['x-desk-name'] → المعامل name
```

## أبسط مثال

```ts
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';

export const DeskName = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): string | undefined => {
    const request = ctx.switchToHttp().getRequest<{
      headers: Record<string, string | undefined>;
    }>();
    return request.headers['x-desk-name'];
  },
);
```

```ts
@Post('refund')
refund(@DeskName() desk: string | undefined): { desk: string | undefined } {
  return { desk };
}
```

إن لصق الحارس الاسم على الترويسة أو أرسلها العميل، تصل السلسلة إلى المعامل. الغياب `undefined` ما زال ممكنًا إن لم يرفض الحارس. الزينة لا ترمي `401` في هذا الدرس.

## شرح المثال

- `_data` حاضر لأن التوقيع ثنائي حتى لو لم تستخدم الوسيط.
- `switchToHttp` لأن الشباك HTTP. سياق RPC لا يملك هذه الترويسة بالشكل نفسه.
- لا `@Injectable` على الثابت. هو دالة زينة لا صنف.
- الاستيراد في المتحكم قيمة لا `import type`، وإلا تختفي الزينة.

## ماذا يحدث داخليًا؟

بعد نجاح الحراس تبني Nest وسائط الدالة. لكل معامل زينة تستدعي المصنع بـ `data` والسياق. الناتج يُمرَّر إلى أنابيب ذلك المعامل إن وُجدت، ثم إلى الدالة. فشل أنبوب على الزينة يمنع الدالة مثل `@Param`.

`ExecutionContext` هنا هو نفسه الذي رآه الحارس، بعد أن يكون اللصق على `request` قد تم. لذلك القراءة ترى `user` الذي وضعه `IdentifyDeskGuard`. إن استُدعيت الزينة في اختبار من غير سياق، تمرر سياقًا مزيفًا إلى المصنع عبر الدالة التي يصدرها Nest للاختبار، أو تستخرج الدالة الداخلية. لا تشغّل الخادم لقراءة ترويسة.

المصنع المتزامن يكفي للترويسة. عمل غير متزامن ثقيل لا يعيش هنا. الحارس `async` مكان الانتظار على رمز، والقيمة الجاهزة تُقرأ في الزينة.

## مثال عملي

`refund` و `audit` كانتا تقرآن `x-desk-name` من `@Req()`. بعد `DeskName` التوقيعان يعرضان `desk: string | undefined`. اختبار المتحكم يمرر الزينة بقيمة `'lead'` عبر طبقة الاختبار الخاصة بـ Nest أو يستدعي المصنع على سياق فيه الترويسة. تغيير اسم الترويسة يتم في ملف واحد. الحارس الذي يرفض الغياب يبقى على المسار حتى لا تعتمد الدالة على `undefined` في الإنتاج.

## أخطاء شائعة

- رمي `UnauthorizedException` من المصنع ومن الحارس معًا برسالتين.
- قراءة `ctx` وتخزين الطلب في `let` على مستوى الملف.
- توقع الفحص بـ `ValidationPipe` العام من غير `validateCustomDecorators`.
- وضع `@DeskName` على الصنف. هي زينة معامل.

## متى نستخدمه؟

لاستخراج يتكرر من طلب HTTP إلى وسيط، باسم يوضح العقد.

## متى لا نستخدمه؟

لقيمة استُخرجت مرة. ولبيانات على الدالة يقرأها الحارس قبل الاستدعاء. تلك `SetMetadata` لا معامل. ولا لاستعلام قاعدة. تلك خدمة بعد أن يصل المعرف وسيطًا عاديًا.

## مقارنة

| | `@Req()` | `createParamDecorator` |
| --- | --- | --- |
| نوع الوسيط | الطلب | ما ترجع المصنع |
| مكان الاستخراج | جسم الدالة | ملف الزينة |
| الأنابيب | نعم | نعم، وفحص الزينة يحتاج خيارًا |
| الإذن | لا | لا |

## تمارين

### تمرين سهل

ما الوسيطان اللذان يستلمهما المصنع؟

<details>
<summary>الحل</summary>

`data` من قوس الزينة، و `ExecutionContext`.

</details>

### تمرين متوسط

لماذا قد يبقى `desk` قيمته `undefined` رغم وجود الزينة؟

<details>
<summary>الحل</summary>

الترينسة غائبة والحارس لم يرفض. الزينة تقرأ فقط. الرفض `401` في `canActivate`.

</details>

### تمرين متقدم

تريد أن يرفض `ValidationPipe` اسم مكتب فارغًا على `@DeskName`. ما الخيار الذي تفوّته الزينة المخصصة افتراضيًا؟

<details>
<summary>الحل</summary>

`validateCustomDecorators: true` على `ValidationPipe`. من غيره يتخطى الأنبوب زينة `createParamDecorator`. ويمكن تمرير الأنبوب على المعامل نفسه كما في صفحة الزينة.

</details>

## أسئلة مراجعة

- من أي حزمة يُستورد `createParamDecorator`؟
- متى يُستدعى المصنع بالنسبة إلى الحارس؟
- هل الزينة تثبت الهوية؟
- ماذا يفعل `_data` إن لم تستخدمه؟

## الخلاصة

المصنع يحول السياق إلى قيمة معامل. `data` وسيط اختياري عند الاستخدام. الإذن والفحص الثقيل يبقيان في الحارس والأنبوب، مع تفعيل صريح إن أردت فحص الزينة المخصصة.

## ماذا بعد؟

الدرس التالي `SetMetadata`: لصق قيمة على الدالة أو الصنف بمفتاح، لا استخراج معامل.

## المصادر الرسمية

- [Custom decorators](https://docs.nestjs.com/custom-decorators)
- [Validation](https://docs.nestjs.com/application/validation)
