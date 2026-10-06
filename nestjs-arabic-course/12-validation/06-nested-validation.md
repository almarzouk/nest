# Nested Validation

## الهدف من هذا الدرس

فحص كائن داخل حقل، لا الحقل الأب فقط.

## المشكلة التي يحلها

`address.city` يصل فارغًا والطلب ينجح لأن الزينة على `AddressDto` لم تُشغَّل. `@IsString()` على الأب لا تمشي أبناء الكائن. TypeScript لا ينقل زينة الداخل إلى الفحص وحده.

## الفكرة ببساطة

على الحقل المتداخل: `@ValidateNested()` من `class-validator` و `@Type(() => AddressDto)` من `class-transformer`. الأولى تشغّل فحص الصنف الداخلي. الثانية تبني نسخة ذلك الصنف حتى تُرى زينته.

## التعريف التقني

```ts
@ValidateNested()
@Type(() => AddressDto)
address: AddressDto;
```

`@ValidateNested()` من غير `@Type` يترك `address` كائنًا عاديًا في كثير من الحالات، فتفشل القيود الداخلية في العمل كما تتوقع. `@Type` من غير `@ValidateNested` يبني النسخة ولا يطلب فحص الأبناء.

للمصفوفة المتداخلة، الصفحة تذكّر أن `TypeScript` لا يصدر بيانات لـ `AddressDto[]`. إما صنف يغلّف القائمة، أو `ParseArrayPipe` مع `items`. لفحص كل عنصر كائن:

```ts
@ValidateNested({ each: true })
@Type(() => AddressDto)
stops: AddressDto[];
```

`each: true` تمشي العناصر. `@Type` ما زال يحدد صنف العنصر.

`transform: true` على الأنبوب حتى تستلم الدالة النسخ المتداخلة لا الكائنات العادية فقط. الفحص يمكن أن ينجح داخليًا قبل أن ترجع النسخة، لكن المعالج الذي يعتمد على `instanceof` يحتاج `transform`.

خيارات `whitelist` تطبق على الداخل حين تُفحص النسخة الداخلية. حقل زائد داخل `address` يُحذف أو يُرفض حسب الخيار العام.

## Mental Model

```text
ReserveSeatDto
  seat          يُفحص مباشرة
  address       @Type يبني AddressDto
                @ValidateNested يفحص city
```

## أبسط مثال

```ts
import { Type } from 'class-transformer';
import { IsInt, IsString, Min, MinLength, ValidateNested } from 'class-validator';

export class AddressDto {
  @IsString()
  @MinLength(2)
  city: string;
}

export class ReserveSeatDto {
  @IsInt()
  @Min(1)
  seat: number;

  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

الجسم `{ "seat": 4, "address": { "city": "ن" } }` يفشل لأن `city` أقصر من 2. الجسم الذي فيه `city: "تونس"` و `seat: 4` يمر من الشكل.

## شرح المثال

- زينة `city` تعيش على `AddressDto` لا على الأب.
- الدالة في `@Type` تؤخر الربط بالصنف.
- فشل الداخل يظهر في أخطاء `children` ثم في `message` حسب `errorFormat`. في وضع `grouped` المفتاح يشبه مسار الخاصية المتداخلة.
- `seat` ما زال يُفحص حتى لو فشل `city`. المكتبة تجمع الأخطاء ما لم تضبط `stopAtFirstError`.

## ماذا يحدث داخليًا؟

`plainToInstance` يرى `@Type` فيني `AddressDto` من الكائن الداخلي. `validate` يرى `@ValidateNested` فيستدعي الفحص على تلك النسخة. أخطاء الداخل تُربط كأبناء لخاصية `address`.

من غير `@Type`، التحويل لا يعرف الصنف الداخلي. القيود المسجّلة على `AddressDto` لا تُربَط بالكائن العادي، فيمر `city` الفارغ أو العددي.

مصفوفة بلا `each: true` لا تمشي العناصر. مصفوفة بلا `@Type` تبقى عناصرها عادية.

`whitelist` على الأنبوب يطبّق الفكرة على `AddressDto` أيضًا: مفتاح `zip` بلا زينة داخل العنوان يُسقط أو يرفض مع `forbidNonWhitelisted`.

## مثال عملي

حجز مقعد مع عنوان فوترة. العقد يرفض مدينة من حرف واحد قبل فتح الحجز. اختبار يرسل `address: { city: 5 }` ويتوقع فشل `@IsString()` على المسار المتداخل. اختبار يحذف `@Type` من الفرع ويرى الطلب ينجح، فيعيد الزينة. هذا الفرق هو الدرس كله.

## أخطاء شائعة

- `@ValidateNested()` فقط.
- استيراد `@Type` من `class-validator`.
- نسيان `each: true` على مصفوفة عناوين.
- توقع فحص `ReserveSeatDto[]` في `@Body()` من غير غلاف أو `ParseArrayPipe`.

## متى نستخدمه؟

كلما كان الحقل كائن عقد أو قائمة كائنات عقد، لا نصًا واحدًا.

## متى لا نستخدمه؟

لقيمة أولية مثل `seat`. الزينة المباشرة تكفي. ولا تحويلًا لمسار URL. ذلك `Parse*`.

## مقارنة

| الزينة | المكتبة | الدور |
| --- | --- | --- |
| `@ValidateNested()` | `class-validator` | شغّل الفحص على الداخل |
| `@Type(() => AddressDto)` | `class-transformer` | ابنِ صنف الداخل |
| `@IsString()` على `city` | `class-validator` | قاعدة الحقل الداخلي |
| `each: true` | خيار الزينة | طبّق الفحص على كل عنصر |

## تمارين

### تمرين سهل

أي زينتين توضعان على حقل `address` حتى تُفحص `city`؟

<details>
<summary>الحل</summary>

`@ValidateNested()` و `@Type(() => AddressDto)`.

</details>

### تمرين متوسط

لماذا قد ينجح الطلب حين تكون `city` عددًا إذا نُسيت `@Type`؟

<details>
<summary>الحل</summary>

الداخل يبقى كائنًا عاديًا بلا زينة مربوطة، ففحص `AddressDto` لا يعمل على `city`.

</details>

### تمرين متقدم

`stops` مصفوفة عناوين. اكتب الزينتين مع الخيار الذي يمشي كل عنصر.

<details>
<summary>الحل</summary>

`@ValidateNested({ each: true })` و `@Type(() => AddressDto)` على الحقل `stops`. كل عنصر يُبنى كـ `AddressDto` ويُفحص.

</details>

## أسئلة مراجعة

- لماذا لا تكفي زينة الأب؟
- من أي حزمة `@Type`؟
- ماذا يفعل `each: true`؟
- أين تظهر أخطاء الداخل في `ValidationError`؟

## الخلاصة

الفحص المتداخل زينتان معًا: `@ValidateNested` تشغّل القواعد، و `@Type` تبني الصنف الذي يحملها. المصفوفة تحتاج `each: true`. من غير البناء يبقى الداخل كائنًا عاديًا لا تراه القيود.

## ماذا بعد؟

الدرس التالي `StandardSchemaValidationPipe`: فحص مخطط Zod أو Valibot أو ArkType عبر خيار `schema` على الزينة.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
- [Pipes](https://docs.nestjs.com/pipes)
