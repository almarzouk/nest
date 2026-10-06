# class-validator

## الهدف من هذا الدرس

كتابة قواعد الحقول بزينة `class-validator` على صنف العقد، واستدعاء `validate` كما يفعل الأنبوب.

## المشكلة التي يحلها

الشروط المتناثرة `if (!email.includes('@'))` تختلف بين المسارات وتُنسى في التحديث المشتق. المكتبة تضع القاعدة فوق الحقل مرة واحدة.

## الفكرة ببساطة

كل قاعدة دالة زينة: `@IsEmail()` و `@IsString()` و `@MinLength()` و `@IsInt()` و `@IsOptional()`. `ValidationPipe` يستدعي `validate` من المكتبة نفسها. تستطيع استدعاء `validate` في اختبار من غير HTTP.

## التعريف التقني

ثبّت `class-validator` مع `class-transformer`. الزينة تُستورد من `class-validator` لا من `@nestjs/common`.

`validate(object)` ترجع `Promise<ValidationError[]>`. المصفوفة الفارغة نجاح. كل خطأ فيه `property` و `constraints`. الأنبوب المدمج يحوّل هذه المصفوفة إلى `BadRequestException`. شكل الرد وخيارات `whitelist` ليست هذا الدرس.

زينة شائعة على عقد عضو:

- `@IsString()` و `@MinLength(n)` و `@MaxLength(n)` للنص.
- `@IsEmail()` للبريد.
- `@IsInt()` و `@Min(n)` للعدد.
- `@IsOptional()` يسمح بغياب الحقل ولا يتخطى قيمة خاطئة إن وُجدت.
- `@IsBoolean()` للعلم.

القواعد تعمل على نسخة الصنف. كائن عادي `{ email: 'a' }` قد لا يُفحص كما تتوقع حتى يمر `plainToInstance`. الأنبوب يفعل ذلك داخليًا. في الاختبار افعل الأمر نفسه أو ابنِ `new CreateMemberDto()` واملأ الحقول.

`class-validator` ليست مهملة. صفحة Nest الحالية ما زالت تبني `ValidationPipe` عليها، وتضيف مسار مخططات بجانبها. لا تحذف الزينة انتظارًا لحذف لم يحدث.

## Mental Model

```text
حقل email
  @IsEmail()  →  constraint
validate(instance) → [] أو [{ property: 'email', constraints }]
```

## أبسط مثال

```ts
import { IsEmail, IsOptional, IsString, MinLength } from 'class-validator';

export class CreateMemberDto {
  @IsString()
  @MinLength(3)
  fullName: string;

  @IsEmail()
  email: string;

  @IsOptional()
  @IsString()
  city?: string;
}
```

```ts
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';
import { CreateMemberDto } from './create-member.dto.js';

const dto = plainToInstance(CreateMemberDto, {
  fullName: 'لم',
  email: 'not-an-email',
});
const errors = await validate(dto);
```

`errors` فيها عنصر لـ `fullName` وعنصر لـ `email`. `city` غائبة و `@IsOptional()` يجعل الغياب مقبولًا.

## شرح المثال

- زينتان على `fullName` تُجمعان. الفشل قد يذكر الطول والنوع معًا إن وصلت قيمة ليست نصًا طويلًا.
- `@IsOptional()` فوق `@IsString()` على `city`. الترتيب المعتاد: الاختياري أولًا ثم القاعدة.
- `plainToInstance` من `class-transformer` يصنع النسخة التي ترى الزينة. الدرس التالي يشرح المكتبة. هنا الاستدعاء لازم حتى لا تفحص كائنًا عاديًا بالخطأ.
- لا `ValidationPipe` في ملف الاختبار. الدالة `validate` هي المحرك.

## ماذا يحدث داخليًا؟

عند تحميل الصنف، كل زينة تسجّل قيدًا في بيانات الخاصية. `validate` يمشي الحقول، يشغّل القيود، ويجمع `ValidationError`. بعض القيود غير متزامنة، لذلك `validate` ترجع وعدًا، ولذلك `transform` في أنبوب التحقق `async`.

الأنبوب المدمج يمرر خيارات `ValidatorOptions` مثل `whitelist` إلى هذه الدالة. من غير خيارات، الحقل الزائد لا يفشل الفحص ولا يُحذف. `validate` تخبرك عن الحقول المزيّنة فقط.

رسالة القيد الافتراضية إنجليزية من المكتبة، مثل أن البريد يجب أن يكون بريدًا. تخصيص الرسالة وسيط في الزينة، وتجميعها في الرد درس شكل الأخطاء.

## مثال عملي

اختبار واحد على `CreateMemberDto`: الجسم `{ fullName: 'هدى', email: 'huda@example.com' }` يعطي مصفوفة فارغة. الجسم الذي ينقصه `@` يعطي خطأ على `email`. مسار `POST` لا يعيد كتابة القاعدة. حين تشتق `UpdateMemberDto` بـ `PartialType`، `@IsEmail()` تبقى على `email` إن أُرسل الحقل.

## أخطاء شائعة

- استيراد `@IsEmail` من حزمة Nest. هو من `class-validator`.
- استدعاء `validate` على كائن عادي بلا `plainToInstance` ثم الاعتقاد أن الزينة معطلة.
- `@IsOptional()` على حقل مطلوب في الإنشاء. الاختياري للتحديث أو للحقل غير الإلزامي فقط.
- كتابة قاعدة البريد في الخدمة أيضًا. مصدر واحد: الزينة.

## متى نستخدمه؟

على حقول عقد الدخل التي يفحصها `ValidationPipe`، وفي اختبار يستدعي `validate` مباشرة.

## متى لا نستخدمه؟

لفحص إعداد البيئة إن اخترت مخططًا هناك، ولاستبدال `ParseUUIDPipe` على معامل نص واحد إن كان الأنبوب المدمج أصرح. ولا تضعها على صنف الرد إلا لحاجة تسلسل لاحقة.

## مقارنة

| الزينة | تقبل | ترفض مثلًا |
| --- | --- | --- |
| `@IsEmail()` | بريدًا نصيًا | `not-an-email` |
| `@MinLength(3)` | نصًا طويلًا كفاية | `"لم"` |
| `@IsInt()` | عددًا صحيحًا | `"3"` كنص |
| `@IsOptional()` | غياب الحقل | لا يعطّل القاعدة إن وُجدت قيمة |

## تمارين

### تمرين سهل

من أي حزمة تُستورد `@IsEmail()`؟

<details>
<summary>الحل</summary>

من `class-validator`.

</details>

### تمرين متوسط

ماذا تعني مصفوفة فارغة من `validate`؟

<details>
<summary>الحل</summary>

كل القيود على النسخة نجحت. الأنبوب في هذه الحالة لا يرمي بسبب الفحص.

</details>

### تمرين متقدم

`city` اختيارية، لكن إن أُرسلت يجب أن تكون نصًا. أي زينتين تضع، وماذا يحدث لو أرسل العميل `"city": 4`؟

<details>
<summary>الحل</summary>

`@IsOptional()` و `@IsString()`. الغياب ينجح. القيمة `4` تفشل قيد النص لأن الاختياري لا يمرر قيمة حاضرة خاطئة.

</details>

## أسئلة مراجعة

- ما اسم دالة المكتبة التي ترجع الأخطاء؟
- لماذا الاختبار يحتاج `plainToInstance`؟
- هل المكتبة مهملة في توثيق Nest الحالي؟
- ماذا يفحص `validate` في حقل بلا زينة؟

## الخلاصة

`class-validator` تضع قيد الحقل على الصنف. `validate` ترجع الأخطاء، و `ValidationPipe` يحوّلها إلى رفض طلب. القواعد تبقى مع `PartialType`. المكتبة مسار مدعوم بجانب المخططات لا بدلًا محذوفًا.

## ماذا بعد؟

الدرس التالي `class-transformer`: تحويل الكائن العادي إلى نسخة الصنف، وهو ما يحتاجه الفحص والتحويل لاحقًا.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
- [class-validator](https://github.com/typestack/class-validator)
