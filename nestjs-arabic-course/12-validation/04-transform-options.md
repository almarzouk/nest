# transform و transformOptions

## الهدف من هذا الدرس

تسليم المعالج نسخة من صنف العقد، وضبط التحويل الذي تمرره `transformOptions` إلى `class-transformer`.

## المشكلة التي يحلها

الفحص قد ينجح والمعالج ما زال يمسك كائنًا عاديًا: `instanceof ReserveSeatDto` خاطئ، والحقل المتداخل ليس نسخة صنفه. مسار واستعلام يصلان نصًا بينما التوقيع يقول `number`.

## الفكرة ببساطة

`transform: true` يعيد إلى الدالة القيمة بعد التحويل: نسخة الصنف للكائن، وتحويلًا للأنواع الأولية في المسار والاستعلام حسب النوع المنعكس. `transformOptions` خيارات `class-transformer` مثل `enableImplicitConversion`.

## التعريف التقني

من صفحة التحقق: الأجسام تصل كائنات عادية. `transform: true` على الأنبوب، محليًا أو عامًا، يفعّل التحويل التلقائي. مع هذا الخيار يحوّل الأنبوب أيضًا أنواعًا أولية في المسار والاستعلام. المثال الرسمي: `@Param('id') id: number` يطبع `typeof id === 'number'` لأن المعامل وصل نصًا والانعكاس قال `Number`.

لا يوجد `ParseStringPipe` لأن المسار والاستعلام نص أصلًا.

`transformOptions` تُمرَّر إلى `plainToInstance`. الخيار الذي تقابله في العقود المتداخلة والحقول:

```ts
new ValidationPipe({
  transform: true,
  transformOptions: {
    enableImplicitConversion: true,
  },
})
```

`enableImplicitConversion` يحوّل حسب `design:type` داخل الكائن، مثل `"4"` على حقل `number`. التحويل الضمني أوسع من تحويل مسار واحد، وقد يجعل قيمة خاطئة تبدو عددًا. فعّله حيث تقبل هذا السلوك، لا كنسخة افتراضية على كل عقد إن كان `@IsInt()` يجب أن يرفض النص.

`@Type(() => Nested)` ما زال لازمًا لنسخة الحقل المتداخل. التحويل الضمني لا يستبدل تلك الزينة لكائن داخلي.

من غير `transform: true`، المعالج قد يستلم الأصل بعد الفحص. النسخة الداخلية استُخدمت للتحقق ثم لم تُرجَع.

## Mental Model

```text
transform: false   الفحص على نسخة، الدالة قد ترى الأصل
transform: true    الدالة ترى الناتج المحوّل، ومنها نسخة الصنف
enableImplicitConversion   "4" قد تصبح 4 داخل الحقل
```

## أبسط مثال

```ts
import { ValidationPipe } from '@nestjs/common';

new ValidationPipe({
  transform: true,
})
```

```ts
@Get(':id')
findOne(@Param('id') id: number): { id: number } {
  return { id };
}
```

مع الخيار، `GET /reservations/4` يجعل `typeof id` يساوي `'number'` كما في مثال الصفحة. من غير الخيار، القيمة تبقى النص `"4"` رغم أن TypeScript قال `number`.

لجسم الحجز:

```ts
@Post()
reserve(@Body() dto: ReserveSeatDto): boolean {
  return dto instanceof ReserveSeatDto;
}
```

مع `transform: true` الناتج `true` بعد نجاح الفحص.

## شرح المثال

- تحويل `id` هنا يعتمد على النوع المنعكس `Number` مع `transform: true`، لا على `ParseIntPipe`. الأنبوب المدمج للتحليل ما زال أصرح حين تريد رسالة «سلسلة رقمية» الخاصة به.
- `instanceof` على الجسم يحتاج إعادة النسخة إلى الدالة، وهذا معنى `transform` للكائنات.
- `transformOptions` ليست مطلوبة لمثال المسار في الصفحة الحالية. أضفها حين تحتاج سلوك `class-transformer` داخل الحقول.
- اجمع `whitelist: true` مع `transform: true` في التطبيق حين تريد نسخة نظيفة. كل خيار يبقى مفهومًا وحده.

## ماذا يحدث داخليًا؟

الأنبوب يستدعي `plainToInstance(metatype, value, transformOptions)`. عند النجاح و `transform: true` تكون هذه النسخة هي قيمة المعامل. عند `transform: false` تبقى النسخة أداة فحص.

للأنواع الأولية في `param` و `query`، الأنبوب يحوّلها حين يكون التحويل مفعّلًا والنوع المنعكس `Number` أو `Boolean`. هذا يحتاج `emitDecoratorMetadata` حتى يُعرف النوع. إغلاق إصدار البيانات يعيد النص.

`enableImplicitConversion` يوسّع الفكرة إلى خصائص الكائن. `"seat": "4"` قد يمر `@IsInt()` بعد أن صار `4`. إن كان العقد يفرّق بين النص والعدد، اترك الخيار مطفأً وأصلح العميل أو استخدم تحويلًا في حقل واحد عبر `@Type(() => Number)` حيث تقصده.

الحقل المتداخل: `@Type` يحدد الصنف. من غيرها `instanceof` على الداخل يفشل رغم `transform: true` على الأب.

## مثال عملي

`GET /shows/:id` في الصالة يريد عددًا داخليًا. فريق يفعّل `transform: true` فيرى `id` عددًا. فريق آخر يبقي `ParseIntPipe` على المعامل لأن رسالة الفشل أوضح. الاثنان صحيحان إن لم يزدوجا على المعامل نفسه بلا داعٍ.

`POST /reservations` يفعّل `transform: true` حتى تسجّل الخدمة `dto instanceof ReserveSeatDto`. لا يفعّل `enableImplicitConversion` لأن نموذجًا يرسل `seat` كنص يجب أن يُرفض ويُصلح في الواجهة.

## أخطاء شائعة

- توقع `instanceof` من غير `transform: true`.
- تفعيل التحويل الضمني ثم اختفاء أخطاء `@IsInt()` على النصوص.
- نسيان `@Type` للمتداخل والاعتقاد أن `transform` يكفي.
- تحويل كل الاستعلامات ضمنيًا بينما `ParseBoolPipe` على علم واحد كان يكفي.

## متى نستخدمه؟

`transform: true` حين تريد نسخة الصنف في الدالة أو تحويل مسار واستعلام أوليًا حسب النوع المعلن. `enableImplicitConversion` لحقل تعرف أن عملاءه يرسلون نصًا لعدد، مع اختبار يثبت التحويل.

## متى لا نستخدمه؟

لا تفعّل التحويل الضمني لإخفاء نموذج مكسور على كل الحقول. ولا تستبدل `ParseUUIDPipe` بتحويل ضمني. UUID يبقى نصًا مفحوصًا.

## مقارنة

| الخيار | الأثر |
| --- | --- |
| بلا `transform` | فحص، والأصل غالبًا هو ما يصل للدالة |
| `transform: true` | النسخة أو القيمة المحوّلة تصل للدالة |
| `enableImplicitConversion` | تحويل خصائص حسب النوع المنعكس |
| `@Type` | صنف الحقل المتداخل |

## تمارين

### تمرين سهل

ماذا يجب أن يرجع `dto instanceof ReserveSeatDto` داخل المعالج بعد `transform: true` ونجاح الفحص؟

<details>
<summary>الحل</summary>

`true`، لأن المعامل هو ناتج التحويل إلى الصنف.

</details>

### تمرين متوسط

لماذا قد يبقى `@Param('id') id: number` نصًا في التشغيل رغم أن البناء نجح؟

<details>
<summary>الحل</summary>

المسار نص. من غير `transform: true` أو `ParseIntPipe` لا يبدّل الأنبوب القيمة. نوع TypeScript لا يعيش على السلك.

</details>

### تمرين متقدم

`seat` يصل `"4"` ويمر `@IsInt()` بعد نشر الإعداد. أي خيار سبّب ذلك، ومتى تطفئه؟

<details>
<summary>الحل</summary>

`transformOptions.enableImplicitConversion`. أطفئه إن كان العقد يرفض النص، وأصلح العميل ليرسل عددًا. أبقه فقط حيث التحويل مقصود ومختبر.

</details>

## أسئلة مراجعة

- ماذا يغيّر `transform: true` للمعالج؟
- إلى أي مكتبة تذهب `transformOptions`؟
- هل التحويل الضمني يغني عن `@Type`؟
- متى يبقى `ParseIntPipe` أوضح؟

## الخلاصة

`transform: true` يسلّم الدالة ناتج التحويل، نسخة صنف أو قيمة أولية محوّلة في المسار والاستعلام. `transformOptions.enableImplicitConversion` يوسّع التحويل إلى الحقول وقد يخفي نصًا كان يجب رفضه. `@Type` يبقى للحقل المتداخل.

## ماذا بعد؟

الدرس التالي شكل رسالة الخطأ: `disableErrorMessages` و `errorFormat` بقيمتيه `list` و `grouped`.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
