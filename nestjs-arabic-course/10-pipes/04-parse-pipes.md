# ParseIntPipe و ParseBoolPipe و ParseUUIDPipe و ParseArrayPipe

## الهدف من هذا الدرس

استخدام الأنابيب الأربعة على مسار واستعلام وجسم، مع الخيارات التي تغيّر الرمز أو شكل القائمة أو إصدار UUID.

## المشكلة التي يحلها

الاستعلام والنمط في العنوان يصلان نصًا. الخدمة التي تجمع أرقام صفحات أو تقارن UUID تنكسر بصمت إن بقي النص، أو ترمي `500` إن استخدمت `parseInt` بلا رفض واضح.

## الفكرة ببساطة

كل أنبوب يُربَط كوسيط ثانٍ في `@Param` أو `@Query` أو `@Body`. النجاح يبدّل النوع. الفشل `400` ما لم تغيّر `errorHttpStatusCode`.

## التعريف التقني

`ParseIntPipe` يقبل سلسلة رقمية صحيحة. `GET /invoices/abc` على `@Param('id', ParseIntPipe)` ينتج:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

`ParseBoolPipe` يقبل القيم المنطقية التي يفهمها الأنبوب، ومنها النصوص `true` و `false` في الاستعلام.

`ParseUUIDPipe` يقبل UUID من الإصدارات التي تذكرها الصفحة، بما فيها Nil و Max، ما لم تمرر `version`. القيم الموثّقة للخيار: `'1'` حتى `'8'`.

`ParseArrayPipe` يفك مصفوفة. لاستعلام مفصول بفواصل: `{ items: Number, separator: ',' }` على `?ids=1,2,3`. لعناصر هي أصناف DTO: `{ items: CreateLineDto }` لأن TypeScript لا يصدر بيانات لأنواع عامة مثل `CreateLineDto[]`، فـ `ValidationPipe` وحده لا يفحص عناصر المصفوفة العارية.

الخيار `optional: true` على أنابيب `Parse*` يسمح بغياب القيمة. من غيره، `null` و `undefined` يُرفضان. `exceptionFactory` يستبدل الاستثناء الذي يُرمى.

تمرير الصنف بلا `new` يكفي حين لا توجد خيارات.

## Mental Model

```text
"4"        ParseIntPipe     → 4
"true"     ParseBoolPipe    → true
uuid نص    ParseUUIDPipe    → النص نفسه إن كان UUID
"1,2,3"    ParseArrayPipe   → [1, 2, 3]
```

## أبسط مثال

```ts
import { ParseIntPipe, ParseUUIDPipe } from '@nestjs/common';

@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number): { id: number } {
  return { id };
}

@Get('ref/:ref')
findByRef(
  @Param('ref', new ParseUUIDPipe({ version: '4' })) ref: string,
): { ref: string } {
  return { ref };
}
```

`/invoices/4` يستلم `id` كعدد. `/invoices/ref/` متبوعًا بـ UUID إصدار 4 يمر. إصدار 1 يُرفض حين يكون الخيار `version: '4'`.

## شرح المثال

- `ParseIntPipe` بلا خيارات يبنيه الإطار.
- `new ParseUUIDPipe({ version: '4' })` نسخة لأن الخيار في البناء.
- النوع في TypeScript `number` أو `string` يطابق ما يرجع الأنبوب. الأنبوب هو الذي يجعل ذلك صادقًا وقت التشغيل.
- غيّر الرمز بـ `new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE })` إن كان العقد `406`. الافتراضي `400` أوضح لمعظم الواجهات.

## ماذا يحدث داخليًا؟

المنصة تمرر النص. الأنبوب يحلله. النجاح يستبدل المعامل. الفشل يرمي قبل الدالة فيُلتقط كأي استثناء أنبوب: فلاتر المسار ثم المتحكم ثم العام.

`ParseArrayPipe` على الجسم مع `items: CreateLineDto` يبني كل عنصر ويفحصه. على الاستعلام يفصل النص عند `separator` ثم يحوّل كل قطعة حسب `items`.

`ParseBoolPipe` على `sort=true` يعطي `true` المنطقي. النص `"true"` في JSON الجسم قد يكون أصلًا منطقيًا إن حلّل `express.json` الجسم. الأنبوب أهم على الاستعلام والمسار حيث كل شيء نص.

هذه الأنابيب لا تزيل حقولًا زائدة من كائن. ذلك `whitelist` في `ValidationPipe`.

## مثال عملي

```ts
import { ParseArrayPipe, ParseBoolPipe, ParseIntPipe } from '@nestjs/common';

@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
  @Query('open', ParseBoolPipe) open: boolean,
  @Query('page', ParseIntPipe) page: number,
): { ids: number[]; open: boolean; page: number } {
  return { ids, open, page };
}
```

`GET /invoices?ids=1,2,3&open=true&page=2` يصل إلى الدالة والمصفوفة أعداد والعلم منطقي والصفحة عدد. `open=yes` يُرفض. `ids` بلا فاصلة تُفهم قطعة واحدة إن كانت رقمًا صالحًا.

## أخطاء شائعة

- `ParseIntPipe` على UUID. الشكل UUID نص يبقى نصًا عبر `ParseUUIDPipe`.
- نسيان `separator` ثم الدهشة أن `"1,2"` لا يصبح مصفوفة.
- توقع فحص عناصر `CreateLineDto[]` من غير `ParseArrayPipe` أو صنف يغلّف المصفوفة. البيانات الوصفية لا تُصدر للمصفوفة العامة.
- `version: '4'` مع معرّفات إصدار 7 قادمة من قاعدة جديدة. الخيار يجب أن يطابق ما تولّده.

## متى نستخدمه؟

على `@Param` و `@Query` و `@Body` حين تكون القاعدة أحد الأشكال الأربعة: عدد صحيح، منطق، UUID، أو قائمة.

## متى لا نستخدمه؟

لنص حر مثل اسم عميل، أو لكائن فيه عدة حقول مزيّنة. الأول يمر بلا أنبوب أو بأنبوب نطاق، والثاني `ValidationPipe`. ولا تستخدمها لختم الطلب.

## مقارنة

| الأنبوب | دخل شائع | خرج ناجح |
| --- | --- | --- |
| `ParseIntPipe` | `"2"` | `2` |
| `ParseBoolPipe` | `"false"` | `false` |
| `ParseUUIDPipe` | UUID نصًا | النص إن طابق الإصدار |
| `ParseArrayPipe` | `"1,2"` أو JSON مصفوفة | مصفوفة عناصر محللة |

## تمارين

### تمرين سهل

ما رمز الحالة الافتراضي حين يفشل `ParseIntPipe` على `abc`؟

<details>
<summary>الحل</summary>

`400` مع رسالة أن سلسلة رقمية مطلوبة.

</details>

### تمرين متوسط

كيف تقصر `ParseUUIDPipe` على الإصدار 7؟

<details>
<summary>الحل</summary>

`new ParseUUIDPipe({ version: '7' })` على المعامل.

</details>

### تمرين متقدم

لماذا يفشل `@Body() lines: CreateLineDto[]` في فحص العناصر مع `ValidationPipe` عام، وما البناء الذي تعطيه صفحة التحقق؟

<details>
<summary>الحل</summary>

TypeScript لا يصدر بيانات للأنواع العامة، فالعناصر لا تُفحص. استخدم `new ParseArrayPipe({ items: CreateLineDto })`، أو صنفًا يغلّف القائمة في حقل يُفحص.

</details>

## أسئلة مراجعة

- ماذا يرجع `ParseIntPipe` عند النجاح؟
- كيف تطلب إصدار UUID بعينه؟
- ماذا يفعل `separator` في `ParseArrayPipe`؟
- لماذا لا يكفي نوع المصفوفة في التوقيع لفحص العناصر؟

## الخلاصة

الأربعة أنابيب تحليل مربوطة على المعامل. العدد والمنطق والقائمة تتغير أنواعها. UUID يبقى نصًا بعد التأكد من شكله. الخيارات تضبط الرمز والإصدار وفاصل القائمة وعنصر المصفوفة.

## ماذا بعد؟

الدرس التالي مقدمة `ValidationPipe`: فحص صنف مزيّن بـ `class-validator`، بلا الغوص في كل الخيارات.

## المصادر الرسمية

- [Pipes](https://docs.nestjs.com/pipes)
- [Validation](https://docs.nestjs.com/application/validation)
