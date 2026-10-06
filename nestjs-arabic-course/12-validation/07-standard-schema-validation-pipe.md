# StandardSchemaValidationPipe

## الهدف من هذا الدرس

فحص الدخل بمخطط يطبّق Standard Schema، مثل Zod و Valibot و ArkType، عبر الأنبوب المدمج.

## المشكلة التي يحلها

عقد الدعوة مكتوب `z.object` لا صنفًا مزيّنًا. `ValidationPipe` يتخطاه لأنه ليس صنف `class-validator`. القواعد تبقى قيمة بلا من يقرأها على الطلب.

## الفكرة ببساطة

سجّل `StandardSchemaValidationPipe` عامًا. على المعامل مرّر المخطط في خيار الزينة: `@Body({ schema: reserveSchema })`. الأنبوب يفحص من يعلن مخططًا ويمرر الباقي. الناتج الافتراضي هو مخرج المخطط بعد التحويل والقيم الافتراضية.

## التعريف التقني

الأنبوب من `@nestjs/common`. الصفحة تثبّت الحزمة التي تختارها، والمثال Zod:

```bash
npm i zod
```

```ts
import { z } from 'zod';

export const reserveSchema = z.object({
  showId: z.string().min(1),
  seat: z.number().int().positive(),
});

export type ReserveInput = z.infer<typeof reserveSchema>;
```

```ts
import { StandardSchemaValidationPipe } from '@nestjs/common';

app.useGlobalPipes(new StandardSchemaValidationPipe());
```

```ts
@Post()
reserve(@Body({ schema: reserveSchema }) dto: ReserveInput): ReserveInput {
  return dto;
}
```

الخيار `schema` يعمل على `@Body()` و `@Query()` و `@Param()`، وعلى `@MessageBody()` و `@Payload()` في النقلات الأخرى. يمكن أن يأتي بعد اسم خاصية: `@Body('seat', { schema })`.

فشل الفحص `400`. الرسائل في مثال الصفحة مسبوقة بمسار الخاصية، مثل `seat: ...`.

الخيارات على بناء الأنبوب:

- `transform` افتراضيًا `true`: الدالة تستلم مخرج المخطط. `false` يمرر الدخل الأصلي بعد نجاح الفحص، ونوعه عندها `z.input`.
- `validateCustomDecorators` افتراضيًا `false`.
- `validateOptions` تُمرَّر إلى `~standard.validate`.
- `errorHttpStatusCode` افتراضيًا `400`.
- `exceptionFactory` تستلم مشكلات المخطط.

المسار والاستعلام نصان. التحويل يتم داخل المخطط، مثل `z.coerce.number()`، لا عبر `ParseIntPipe` إلا إذا اخترت ذلك الأسلوب.

خصائص غير المعرّفة قرار المخطط: `z.object` يسقطها، `z.strictObject` يرفضها، `z.looseObject` يبقيها. لا `whitelist` على هذا الأنبوب.

Valibot و ArkType الكائن نفسه في خيار `schema`. نص الخطأ يأتي من المكتبة فيختلف لفظه.

## Mental Model

```text
معامل بلا schema     يمر
معامل مع schema      ~standard.validate
  فشل → 400
  نجاح → مخرج المخطط إن transform: true
```

## أبسط مثال

```ts
import { z } from 'zod';

export const listQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  open: z.coerce.boolean().optional(),
});

export type ListQuery = z.infer<typeof listQuerySchema>;
```

```ts
@Get()
list(@Query({ schema: listQuerySchema }) query: ListQuery): ListQuery {
  return query;
}
```

`GET /reservations?page=2` يجعل `page` العدد `2`. غياب `page` يجعلها `1` بسبب `default`. النص في الاستعلام يُجبر عبر `coerce` قبل القاعدة.

## شرح المثال

- المخطط مصدر الحقيقة. `ListQuery` نوع مشتق لا يُفحص وحده.
- `default` و `coerce` آثار تحويل داخل المخطط، وتصل إلى الدالة لأن `transform` على الأنبوب افتراضيًا `true`.
- لا زينة `@IsInt()` بجانب `page` في هذا الأسلوب.
- الربط العام آمن كما تقول الصفحة: من لا يعلن `schema` لا يُفحص بهذا الأنبوب.

## ماذا يحدث داخليًا؟

الأنبوب يبحث عن مخطط على بيانات المعامل. إن وجده يستدعي واجهة Standard Schema. المشكلات تُحوَّل إلى استثناء HTTP. النجاح يعيد المخرج، وفيه القيم الافتراضية والنصوص المشذّبة إن عرّفها المخطط، كما في مثال الصفحة لـ `search` بعد `trim`.

`transform: false` يوقف تسليم المخرج ويبقي الدخل بعد أن نجح الفحص. استخدم `z.input<typeof schema>` عندها.

`exceptionFactory` تستلم `issues` وفيها `message` و `path`. الصفحة تبني منها `422` إن أردت رمزًا آخر. أبقِ الرمز في عائلة أخطاء العميل.

مصفوفة: `z.array(reserveSchema)` على `@Body({ schema })` تفحص كل عنصر. رسالة العنصر تُسبق بفهرسه في مثال الصفحة.

## مثال عملي

صالة جديدة تكتب الحجوزات بمخطط Zod واحد. `POST` يستخدم `reserveSchema`. `GET` يستخدم `listQuerySchema` مع `coerce` و `default`. اختبار يرسل `seat: 0` ويتوقع `400` ونصًا يذكر `seat`. اختبار يرسل `page` غائبًا ويتوقع `1` في القيمة التي تستلمها الدالة. عقد الصنف القديم لمسار آخر لا يُمس في هذا الدرس.

## أخطاء شائعة

- تمرير المخطط إلى `ValidationPipe`.
- نسيان خيار `{ schema }` ثم الدهشة أن كل جسم يمر رغم وجود أنبوب عام.
- توقع `whitelist` هنا. سلوك المفاتيح الزائدة يُكتب في المخطط.
- `z.infer` كمعامل `@Body()` من غير المخطط في الزينة.

## متى نستخدمه؟

حين يكون العقد قيمة مخطط، وتريد الفحص والتحويل الافتراضي من المكتبة نفسها على `@Body` أو `@Query` أو `@Param`.

## متى لا نستخدمه؟

لصنف عليه زينة `class-validator` فقط. ذلك `ValidationPipe`. ولا تحوّل فشل المخطط إلى `500`.

## مقارنة

| | هذا الأنبوب | `ValidationPipe` |
| --- | --- | --- |
| مصدر القاعدة | كائن مخطط | زينة على صنف |
| الربط على المعامل | `{ schema }` | نوع الصنف |
| مفاتيح زائدة | سلوك المخطط | `whitelist` |
| تحويل الاستعلام | `z.coerce` ونحوه | `transform` و `Parse*` |

## تمارين

### تمرين سهل

أين يُعلَّق `reserveSchema` حتى يفحصه الأنبوب العام؟

<details>
<summary>الحل</summary>

في خيار الزينة: `@Body({ schema: reserveSchema })`.

</details>

### تمرين متوسط

لماذا `GET` بلا `page` يعيد العدد `1` في المثال؟

<details>
<summary>الحل</summary>

المخطط يعرّف `default(1)`، وخيار `transform` على الأنبوب افتراضيًا `true` فيصل مخرج المخطط إلى الدالة.

</details>

### تمرين متقدم

تريد رفض المفتاح `role` على مخطط Zod، لا إسقاطه. أي شكل كائن تستخدم، ولماذا لا تضبط `forbidNonWhitelisted`؟

<details>
<summary>الحل</summary>

`z.strictObject`. المفاتيح الزائدة قرار المخطط. `forbidNonWhitelisted` خيار `ValidationPipe` ولا يغيّر هذا الأنبوب.

</details>

## أسئلة مراجعة

- أي مكتبات تذكرها الصفحة كمخططات متوافقة؟
- ماذا يمر من المعاملات التي بلا `schema`؟
- ما افتراض `transform` على هذا الأنبوب؟
- كيف يُجبر نص الاستعلام إلى عدد داخل Zod؟

## الخلاصة

`StandardSchemaValidationPipe` يفحص المخطط المعلّق على الزينة ويعيد مخرجه افتراضيًا. Zod و Valibot و ArkType تدخل من الخيار نفسه. من لا يعلن مخططًا لا يُمس. خيارات التبييض مكانها المخطط لا أنبوب الأصناف.

## ماذا بعد؟

الدرس التالي يجمع الأنبوبين في تطبيق واحد ويثبت أن `class-validator` ليست مسارًا ملغى.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
