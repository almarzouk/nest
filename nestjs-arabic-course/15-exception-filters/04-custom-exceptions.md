# Custom Exceptions

## الهدف من هذا الدرس

إنشاء فئة تمتد من `HttpException` تحمل رمزًا وجسمًا ثابتين لخطأ مجال التذاكر.

## المشكلة التي يحلها

`ConflictException` نصه يتغير بين الملفات: مرة «مبيع» ومرة «مقفل». الواجهة تريد مفتاح `code` ثابتًا مثل `SEAT_TAKEN` لتترجم الجملة. النص الحر لا يكفي.

## الفكرة ببساطة

صنف `SeatTakenException extends HttpException` يبني الجسم `{ code, seat }` مع `HttpStatus.CONFLICT`. الخدمة ترمي الصنف. الفلتر المدمج يسلسل الجسم لأن الأب `HttpException`.

## التعريف التقني

الوراثة من `HttpException` تبقي `getStatus()` و `getResponse()` وتُبقي الاستثناء ضمن العائلة التي لا تُعامل كعطل مجهول.

```ts
super({ code: 'SEAT_TAKEN', seat }, HttpStatus.CONFLICT, { cause });
```

الكائن هو الجسم كله، كما في درس الأب: لن يظهر `statusCode` إلا إذا أضفته إلى الكائن. إن كان عقد العميل عندكم `statusCode` زائد `code` فضعهما معًا داخل الكائن.

لا تمتد من `Error` فقط ثم تتوقع `409`. من غير الأب يصبح الرد `500`.

الفئة لا تفحص الزينة ولا تفتح قاعدة. هي اسم للشرط الذي اكتشفته الخدمة. الرسالة للبشر يمكن أن تبقى حقل `message` داخل الكائن بجانب `code`.

`@Catch(SeatTakenException)` لاحقًا يخص هذا النوع إن أردت سجلًا إضافيًا. من غيره يكفي المدمج.

## Mental Model

```text
الخدمة اكتشفت المقعد مبيعًا
  throw new SeatTakenException(seat)
  الجسم { code: 'SEAT_TAKEN', seat }
  الرمز 409
```

العميل يتفرع على `code` لا على نص إنجليزي.

## أبسط مثال

```ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class SeatTakenException extends HttpException {
  constructor(seat: number, cause?: unknown) {
    super(
      { code: 'SEAT_TAKEN', seat, message: `Seat ${seat} is already sold` },
      HttpStatus.CONFLICT,
      cause ? { cause } : undefined,
    );
  }
}
```

```ts
if (this.sold.has(key)) {
  throw new SeatTakenException(seat);
}
```

الرد:

```json
{
  "code": "SEAT_TAKEN",
  "seat": 4,
  "message": "Seat 4 is already sold"
}
```

## شرح المثال

- `code` ثابت يمكن للواجهة أن تقارنه.
- `seat` يعيد القيمة التي رُفضت من غير تحليل الجملة.
- `cause` اختياري ولا يخرج في JSON.
- الرمز ما زال `409` لأن المعنى تعارض حالة لا شكل تالف.

## ماذا يحدث داخليًا؟

`super` يمرر الجسم والرمز إلى `HttpException`. عند الصعود يرى الفلتر المدمج فئة معروفة، فيكتب `getStatus()` و `getResponse()`. لا حاجة إلى `@Catch` حتى يظهر `code`.

فلتر `@Catch()` فارغ يمسك هذا الصنف أيضًا لأنه يمسك كل شيء، وقد يضيع `code` إن استبدلت الجسم بـ `{ statusCode: 500 }`. الدرس عن `@Catch` يحذّر من ذلك. هنا الجسم الصحيح يعيش على الاستثناء نفسه.

الاختبار يستدعي الخدمة ويتوقع أن الخطأ `instanceof SeatTakenException` وأن `getStatus()` يساوي `409`. لا تحتاج HTTP لثبات العقد، واختبار HTTP يتأكد أن الجسم خرج كما هو.

## مثال عملي

واجهة الشباك تترجم `SEAT_TAKEN` إلى جملة عربية وتلوّن المقعد. تغيير `message` الإنجليزية لا يكسر الترجمة. مسار ثانٍ يرمي الصنف نفسه من إلغاء مزدوج إن أضفت `code` آخر في صنف ثانٍ `TicketAlreadyCancelledException` بدل فرع نصي داخل فئة واحدة. صنف لكل حالة لها مفتاح ثابت.

## أخطاء شائعة

- الوراثة من `Error` ونسخ `status` كحقل عادي لا يقرأه الفلتر المدمج.
- جسم مختلف في كل `throw new SeatTakenException` عبر وسيط حر يغيّر المفاتيح.
- وضع اسم جدول SQL في `code`.
- تحويل الفئة إلى `500` في فلتر عام لاحق.

## متى نستخدمه؟

حين يحتاج العميل مفتاحًا ثابتًا أو حقولًا غير الرسالة، والحالة تتكرر في أكثر من مسار.

## متى لا نستخدمه؟

لمرة واحدة فيها نص كافٍ: `new NotFoundException('...')` أخف. ولا تستبدل الفئة أنبوب `400`.

## مقارنة

| | `ConflictException` بنص | `SeatTakenException` |
| --- | --- | --- |
| الرمز | `409` | `409` |
| الجسم | `message` غالبًا | `code` و `seat` و `message` |
| تمييز النوع | عام | `instanceof` خاص |
| التكرار | جملة قد تتفاوت | بناء واحد |

## تمارين

### تمرين سهل

من أي صنف تمتد فئة المجال حتى يكتب Nest الرمز الصحيح؟

<details>
<summary>الحل</summary>

من `HttpException`، أو من فئة جاهزة إن كان ذلك يغطي الجسم. الامتداد المباشر من `Error` يعطي `500`.

</details>

### تمرين متوسط

لماذا تضع `code: 'SEAT_TAKEN'` بدل الاعتماد على `message` فقط؟

<details>
<summary>الحل</summary>

النص يتغير مع اللغة والصياغة. المفتاح ثابت تقارنه الواجهة والاختبار.

</details>

### تمرين متقدم

تريد `statusCode` و `code` معًا في JSON. ماذا تغيّر في `super`؟

<details>
<summary>الحل</summary>

تضع المفتاحين داخل كائن الجسم، مثل `{ statusCode: HttpStatus.CONFLICT, code: 'SEAT_TAKEN', seat }`، لأن الكائن يستبدل الجسم الافتراضي ولا يدمجه.

</details>

## أسئلة مراجعة

- لماذا لا يكفي `extends Error`؟
- ماذا يفعل الفلتر المدمج من غير `@Catch` على فئتك؟
- أين يذهب `cause`؟
- متى تبقى الفئة الجاهزة أخف؟

## الخلاصة

فئة المجال تثبت الرمز والجسم فوق `HttpException`. العميل يتفرع على `code`. الفلتر المدمج يسلسل ما مررته إلى `super` ما لم يستبدله فلتر لاحق بجسم أعمى.

## ماذا بعد؟

الدرس التالي واجهة الفلتر: `catch(exception, host)` حين لا يكفي تسلسل المدمج.

## المصادر الرسمية

- [Exception filters](https://docs.nestjs.com/exception-filters)
