# class-transformer

## الهدف من هذا الدرس

تحويل كائن JSON العادي إلى نسخة من صنف العقد، وتحويل الأنواع المتداخلة بـ `@Type`.

## المشكلة التي يحلها

بعد `express.json()` الجسم `{ desk: "3" }` كائن عادي. `instanceof OpenDeskDto` خاطئ. زينة الفحص المتداخل لا تعمل على كائن داخلي بقي عاديًا. المكتبة تصنع النسخة التي تنتظرها الزينة.

## الفكرة ببساطة

`plainToInstance(Cls, plain)` ترجع نسخة `Cls`. `@Type(() => Nested)` يخبر المكتبة بأي صنف تبني الحقل المتداخل. `@Expose()` و `@Exclude()` يسميان ما يدخل التحويل حين تطلب استراتيجية تعرض الحقول المكشوفة فقط.

## التعريف التقني

الحزمة `class-transformer`، والدالة التي يستخدمها توثيق Nest هي `plainToInstance`. الاسم الأقدم `plainToClass` ما زال معروفًا. التوثيق الحالي يكتب `plainToInstance`.

`ValidationPipe` يستدعيها داخليًا قبل `validate`. خيار `transform: true` على الأنبوب يعيد النسخة إلى المعالج بدل الكائن الأصلي. خيار `transformOptions` يُمرَّر إلى هذه المكتبة، ومنه `enableImplicitConversion`. تفاصيل الخيارين فصل التحقق. هنا سلوك المكتبة وحدها.

`@Type(() => AddressDto)` من `class-transformer` لازمة للحقل المتداخل حتى لو وُجد `@ValidateNested()` من `class-validator`. الفحص المتداخل درس في الفصل التالي. هذا الدرس يعلّم الزينة التي تبني النوع.

`@Expose()` يُظهر الحقل عند `excludeExtraneousValues: true`. `@Exclude()` يخفي الحقل. هذا مفيد لعقد خرج لاحقًا مع المعترض. لا تعتمد عليه اليوم بدل `MemberResponseDto` الذي يبني كائنًا صريحًا، لكن اعرف أين تعيش الزينتان.

## Mental Model

```text
plain { desk: "3", address: { city: "تونس" } }
        │  plainToInstance + @Type
        ▼
OpenDeskDto { desk: "3", address: AddressDto }
```

من غير `@Type`، `address` تبقى كائنًا عاديًا داخل النسخة.

## أبسط مثال

```ts
import { Type } from 'class-transformer';
import { IsInt, IsString, ValidateNested } from 'class-validator';

export class AddressDto {
  @IsString()
  city: string;
}

export class OpenDeskDto {
  @IsInt()
  desk: number;

  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

```ts
import { plainToInstance } from 'class-transformer';
import { OpenDeskDto } from './open-desk.dto.js';

const dto = plainToInstance(OpenDeskDto, {
  desk: 4,
  address: { city: 'تونس' },
});
```

`dto instanceof OpenDeskDto` صحيح، و `dto.address instanceof AddressDto` صحيح بسبب `@Type`.

## شرح المثال

- `plainToInstance` لا يتحقق من القواعد. يبني النسخة. `validate` تأتي بعده.
- `@Type(() => AddressDto)` دالة حتى لا تدور الاعتماديات عند التحميل.
- `desk` وصل عددًا في المثال. النص `"4"` لا يصبح عددًا إلا بتحويل ضمني لاحقًا. لا تفترض أن المكتبة تبدّل كل النصوص.
- `@ValidateNested()` مذكورة حتى لا تُنسى بجانب `@Type` حين تفحص. معناها في فصل التحقق.

## ماذا يحدث داخليًا؟

المكتبة تقرأ بيانات الزينة وتبني `new OpenDeskDto()` ثم تنسخ القيم. للحقل الذي عليه `@Type` تعيد الكرة بصنف `AddressDto`. الحقول التي لا تعرفها تُنسخ أو تُسقط حسب الخيارات. `excludeExtraneousValues: true` يسقط ما ليس `@Expose()`.

الأنبوب عند `transform: false` قد يستخدم النسخة للفحص فقط ويعيد الأصلي إلى الدالة. عند `transform: true` تصل النسخة إلى المعالج، فيعمل `instanceof`. الاختبار الذي يعتمد على النسخة يجب أن يمر من `plainToInstance` أو من أنبوب بالخيار نفسه.

`enableImplicitConversion` يحوّل حسب `design:type`، مثل نص إلى عدد على حقل `number`. هو واسع وقد يخفي خطأ نموذج. ستُفتح مخاطره في درس `transformOptions`. هنا لا تفعّله في كل عقد.

## مثال عملي

طلب فتح مكتب:

```json
{ "desk": 4, "address": { "city": "تونس" } }
```

بعد `plainToInstance` تستطيع خدمة خفيفة أن تقرأ `dto.address.city` وهي تعلم أن `address` نسخة معروفة. طلب فيه `"address": { "city": 1 }` يُبنى كـ `AddressDto` ثم يفشل `@IsString()` عند `validate`. من غير `@Type` قد يتخطى الفحص المتداخل المدينة لأن الداخل ليس نسخة الصنف.

## أخطاء شائعة

- `@Type(() => AddressDto)` من `class-validator`. هي من `class-transformer`.
- نسيان `@Type` والإبقاء على `@ValidateNested` فقط.
- تفعيل التحويل الضمني لكل الحقول ثم قبول `"desk": "4"` من حيث لا تريد.
- استخدام `@Exclude()` على الدخل وانتظار أن `whitelist` قد تم. الإخفاء هنا استراتيجية تحويل، و `whitelist` خيار أنبوب.

## متى نستخدمه؟

حين تبني نسخة عقد في اختبار، وحين يكون الحقل كائنًا متداخلًا يحتاج صنفًا، وحين تريد أن المعالج يستلم `instanceof` عبر `transform: true`.

## متى لا نستخدمه؟

لتحويل `"4"` في مسار URL. `ParseIntPipe` أصرح. ولحذف أسرار الرد إن كان بناء `MemberResponseDto` أوضح من زينة إخفاء على الكيان.

## مقارنة

| | `plainToInstance` | `validate` |
| --- | --- | --- |
| المكتبة | `class-transformer` | `class-validator` |
| الناتج | نسخة صنف | قائمة أخطاء |
| الحقل المتداخل | `@Type` | `@ValidateNested` |

## تمارين

### تمرين سهل

أي دالة تحوّل الكائن العادي إلى نسخة `OpenDeskDto`؟

<details>
<summary>الحل</summary>

`plainToInstance(OpenDeskDto, plain)`.

</details>

### تمرين متوسط

لماذا `@Type(() => AddressDto)` دالة لا إشارة مباشرة إلى الصنف في بعض الملفات؟

<details>
<summary>الحل</summary>

الدالة تؤخر قراءة الصنف حتى وقت التحويل، فتتفادى دورة استيراد حين يشير ملفان كل منهما إلى الآخر.

</details>

### تمرين متقدم

`dto.address instanceof AddressDto` يساوي `false` بعد التحويل. ما الزينة الناقصة؟

<details>
<summary>الحل</summary>

`@Type(() => AddressDto)` على الحقل `address` في `OpenDeskDto`. من غيرها تبقى `address` كائنًا عاديًا.

</details>

## أسئلة مراجعة

- ما الفرق بين بناء النسخة وفحصها؟
- من أي حزمة `@Type`؟
- ماذا يغيّر `excludeExtraneousValues`؟
- متى يستلم المعالج النسخة لا الكائن الأصلي؟

## الخلاصة

`class-transformer` تصنع نسخة الصنف من JSON. `@Type` تبني المتداخل. الفحص مكتبة أخرى تأتي بعد النسخة. تحويل المسار النصي يبقى لأنابيب `Parse*`.

## ماذا بعد؟

الدرس التالي لمحة عن كائن مخطط، مثل Zod، قبل أن يفحصه `StandardSchemaValidationPipe` في فصل التحقق.

## المصادر الرسمية

- [Pipes](https://docs.nestjs.com/pipes)
- [Validation](https://docs.nestjs.com/application/validation)
- [class-transformer](https://github.com/typestack/class-transformer)
