# Built-in Pipes

## الهدف من هذا الدرس

معرفة قائمة الأنابيب المدمجة في `@nestjs/common` ومتى تختار اسمًا منها بدل كتابة صنف.

## المشكلة التي يحلها

كل مشروع يعيد كتابة تحويل العدد والمنطق والقيمة الافتراضية. الأسماء المدمجة موحّدة الرسالة والرمز، والفريق يقرأها في توقيع الدالة.

## الفكرة ببساطة

القائمة جاهزة. هذا الدرس خريطة. الدرس التالي يفتح أربعة أنابيب تحليل بالتفصيل. `ValidationPipe` له مقدمة بعده ثم فصل كامل. `DefaultValuePipe` يُذكر هنا لأنه يسبق أنبوب التحليل على المعامل نفسه.

## التعريف التقني

الصفحة تسرد ما يُصدَّر من `@nestjs/common`:

- `ValidationPipe` و `StandardSchemaValidationPipe` للتحقق من كائن أو مخطط.
- `ParseIntPipe` و `ParseFloatPipe` و `ParseBoolPipe` و `ParseArrayPipe` و `ParseUUIDPipe` و `ParseEnumPipe` و `ParseDatePipe` للتحليل.
- `DefaultValuePipe` يضع قيمة حين يكون المعامل `null` أو `undefined`.
- `ParseFilePipe` لملفات، وفصله مع الرفع لاحقًا.

أنابيب `Parse*` تتوقع قيمة معرّفة. `null` أو `undefined` يرميان إلا إذا ضبطت `optional: true` على أنبوب التحليل. لذلك تأتي `DefaultValuePipe` قبلها في قائمة الزينة حين يكون الاستعلام اختياريًا.

تمرير الصنف `ParseIntPipe` يترك البناء للإطار. `new ParseIntPipe({ errorHttpStatusCode })` حين تريد رمزًا غير `400`.

## Mental Model

```text
قيمة ناقصة؟     DefaultValuePipe ثم Parse*
نص عدد صحيح؟    ParseIntPipe
نص منطقي؟       ParseBoolPipe
UUID؟           ParseUUIDPipe
قائمة؟          ParseArrayPipe
كائن بزينة؟     ValidationPipe لاحقًا
مخطط Zod؟       StandardSchemaValidationPipe في فصل التحقق
```

## أبسط مثال

```ts
import { DefaultValuePipe, ParseIntPipe } from '@nestjs/common';

@Get()
list(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
): { page: number } {
  return { page };
}
```

`GET /invoices` بلا `page` يجعل `page` العدد `1`. `GET /invoices?page=3` يجعله `3`. `GET /invoices?page=abc` يرمي قبل الدالة برسالة أن سلسلة رقمية مطلوبة، والرمز `400`.

## شرح المثال

- الترتيب داخل المعامل من اليسار إلى اليمين: الافتراضي أولًا ثم التحليل. الصفحة تضع `DefaultValuePipe` قبل `Parse*` حتى تُملأ القيمة الناقصة قبل أن يرفضها التحليل.
- الرقم `1` هنا اختيار واجهة الفواتير، لا ثابت في الإطار. مثال الصفحة يستخدم `0` في مسار آخر. المعنى أنك أنت صاحب الافتراضي.
- لا صنف محلي. السلوك موثّق ورسالته معروفة.
- `ParseFloatPipe` و `ParseEnumPipe` و `ParseDatePipe` تُربَط بالطريقة نفسها حين يكون المعامل عددًا عشريًا أو قيمة تعداد أو تاريخًا. لا تُفتح بالتفصيل في الدرس التالي لأن عنوانه أربعة أنابيب محددة.

## ماذا يحدث داخليًا؟

القيمة الاستعلام تصل نصًا. `DefaultValuePipe` إن رأت `undefined` أو `null` ترجع القيمة التي بنيتها بها. وإلا ترجع النص كما هو. `ParseIntPipe` يحوّل النص الصالح إلى عدد صحيح ويرمي إن فشل.

الأنبوب المدمج يرمي داخل منطقة الاستثناءات، فالمعالج لا يعمل. ربط الصنف يمكّن الحقن إن احتاجه أنبوب مخصص، أما `new DefaultValuePipe(1)` فنسخة لأن الخيار في البناء.

`ValidationPipe` و `StandardSchemaValidationPipe` في القائمة لأنهما مدمجان، لكن قواعدهما تعيش على صنف DTO أو على مخطط. هذا الدرس لا يضبط `whitelist` ولا `schema`.

`ParseFilePipe` اسم في القائمة حتى لا تكتبه بنفسك عند رفع ملف لاحقًا.

## مثال عملي

قائمة فواتير: `page` عبر `DefaultValuePipe` و `ParseIntPipe`. رمز الفاتورة في مسار آخر عبر `ParseUUIDPipe` إن كان المعرّف UUID. علم `paid` عبر `ParseBoolPipe`. لا تكتب ثلاثة أصناف محلية لهذه الثلاث. محلل تاريخ الاستحقاق، إن احتجته، `ParseDatePipe` بالربط نفسه.

## أخطاء شائعة

- وضع `ParseIntPipe` قبل `DefaultValuePipe` على معامل ناقص. التحليل يرى الفراغ ويرمي قبل أن يُملأ الافتراضي.
- نسيان `optional: true` حين يكون الحقل غائبًا ولا تريد افتراضيًا ولا رفضًا.
- إعادة تنفيذ `ParseIntPipe` في ملف محلي «لأن القائمة طويلة». الاسم المدمج جزء من عقد القراءة.
- فتح خيارات `ValidationPipe` في هذا الدرس. الخريطة هنا، والخيارات في فصلها.

## متى نستخدمه؟

كلما طابق المعامل اسمًا في القائمة: عدد، منطق، UUID، مصفوفة، افتراضي، أو لاحقًا كائن بزينة.

## متى لا نستخدمه؟

لقاعدة خاصة بنطاقك مثل `INV-18`. ذلك أنبوب مخصص في درس لاحق. ولشكل الرد. ولختم الطلب.

## مقارنة

| الاسم | أثر مختصر |
| --- | --- |
| `DefaultValuePipe` | يملأ `null` و `undefined` |
| `ParseIntPipe` | نص إلى عدد صحيح أو `400` |
| `ParseBoolPipe` | نص إلى `boolean` |
| `ParseUUIDPipe` | يتأكد أن النص UUID |
| `ParseArrayPipe` | يفك قائمة |
| `ValidationPipe` | يفحص صنفًا مزينًا |

## تمارين

### تمرين سهل

من أي حزمة تستورد `ParseIntPipe`؟

<details>
<summary>الحل</summary>

من `@nestjs/common`.

</details>

### تمرين متوسط

`paid` استعلام اختياري، وغيابه يعني `false`. أي أنبوبين، وبأي ترتيب؟

<details>
<summary>الحل</summary>

`new DefaultValuePipe(false)` ثم `ParseBoolPipe` على `@Query('paid', ...)`.

</details>

### تمرين متقدم

لماذا تبقى `ParseFilePipe` و `StandardSchemaValidationPipe` في الخريطة رغم أن هذا الفصل لا يشرحهما؟

<details>
<summary>الحل</summary>

لأنهما من القائمة المدمجة نفسها. معرفتهما تمنع كتابة بديل محلي. رفع الملفات له فصل، والمخططات فصل التحقق. الخريطة تشير، والتفصيل في عنوانه.

</details>

## أسئلة مراجعة

- ماذا يحدث لـ `ParseIntPipe` حين تصل القيمة `undefined` بلا `optional`؟
- لماذا يسبق `DefaultValuePipe` أنبوب التحليل؟
- اذكر ثلاثة أنابيب `Parse*` غير الأربعة التي سيشرحها الدرس التالي.
- أين تعيش خيارات `whitelist`؟

## الخلاصة

`@nestjs/common` يصدّر أنابيب تحليل وافتراضي وتحقق. اختر الاسم المدمج حين يطابق القاعدة. الافتراضي يسبق التحليل على المعامل نفسه. القواعد الخاصة بالنطاق تُكتب صنفًا لاحقًا.

## ماذا بعد؟

الدرس التالي يفتح `ParseIntPipe` و `ParseBoolPipe` و `ParseUUIDPipe` و `ParseArrayPipe`: الخيارات والرسائل.

## المصادر الرسمية

- [Pipes](https://docs.nestjs.com/pipes)
- [Validation](https://docs.nestjs.com/application/validation)
