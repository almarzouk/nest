# Custom Pipes

## الهدف من هذا الدرس

بناء أنبوب نطاق يطبّق `PipeTransform` حين لا تغطي الأسماء المدمجة القاعدة.

## المشكلة التي يحلها

رمز قيد المكتبة `HLD-240` ليس عددًا ولا UUID ولا منطقًا. أنبوب عام للتحقق من أصناف لا يعبّر عن مسار `@Param('hold')` بنص واحد. القاعدة تستحق صنفًا باسمها.

## الفكرة ببساطة

صنف `@Injectable()` فيه `transform`. يقرأ القيمة و `metadata` إن لزم. يرجع نصًا مطبّعًا أو يرمي `BadRequestException`.

## التعريف التقني

```ts
export interface PipeTransform<T = any, R = any> {
  transform(value: T, metadata: ArgumentMetadata): R | Promise<R>;
}
```

`T` نوع الداخل و `R` نوع الخارج. `ArgumentMetadata` فيها `type` و `metatype` و `data` و `schema`.

استخدم `metadata.data` لتعرف اسم المعامل حين يخدم الأنبوب أكثر من زينة، و `metadata.type` لترفض استخدامه على `body` إن كان مكتوبًا لمسار.

الربط على المعامل:

```ts
@Param('hold', HoldCodePipe) hold: string
```

تمرير الصنف يتيح الحقن. `new HoldCodePipe('HLD')` إن أردت بادئة من البناء وبلا حاوية.

لا تعِد كتابة `ParseIntPipe`. الصفحة تبني نسخة تعليمية ثم تقول إن المدمج أغنى. الأنبوب المخصص لما يخص النطاق فقط.

## Mental Model

```text
" hld-240 " → trim → حالة موحّدة → اختبار النمط → "HLD-240"
"240"       → BadRequestException
```

المعالج لا يرى المسافات ولا الحالة الصغيرة.

## أبسط مثال

```ts
import {
  BadRequestException,
  Injectable,
  type PipeTransform,
} from '@nestjs/common';

const HOLD = /^HLD-\d{3,}$/;

@Injectable()
export class HoldCodePipe implements PipeTransform<string, string> {
  transform(value: string): string {
    const code = value.trim().toUpperCase();
    if (!HOLD.test(code)) {
      throw new BadRequestException('hold must look like HLD-240');
    }
    return code;
  }
}
```

```ts
@Get(':hold')
find(@Param('hold', HoldCodePipe) hold: string): { hold: string } {
  return { hold };
}
```

## شرح المثال

- `trim` و `toUpperCase` تحويل. اختبار `HOLD` تحقق. الأثر الذي يراه المعالج نص مطبّع، والرمي عند الكسر.
- النمط ثابت في الملف لا داخل المتحكم.
- الصنف بلا `constructor` ما زال `@Injectable()` حتى يبقى الربط بالصنف متسقًا مع بقية الأنابيب.
- لا قراءة لقاعدة البيانات. `HLD-240` قد يكون غير موجود، وهذا `404` من الخدمة لا `400` من شكل الرمز.

## ماذا يحدث داخليًا؟

Nest يستدعي `transform` بالقيمة الخام للوسيط `hold`. النص الراجع يُحقن في المعامل `hold`. الاستثناء يمنع الدالة ويمر بفلاتر السياق.

إن حقنت `HoldPrefix` من الوحدة، مرّر الصنف لا `new HoldCodePipe()`. النسخة التي تبنيها أنت لا تسأل الحاوية، مثل الحارس والأنبوب العام المبني خارج الوحدة.

`metadata.schema` يبقى فارغًا هنا. لا تعلّق مخطط Zod على هذا الصنف. ذلك مسار `StandardSchemaValidationPipe`.

الاختبار يستدعي `new HoldCodePipe().transform(' hld-240 ')` ويتوقع `HLD-240`، ويستدعي `'nope'` ويتوقع الرمي. لا حاجة إلى تشغيل HTTP لقاعدة النمط.

## مثال عملي

مكتب الحجوزات يقبل `GET /holds/hld-240` من نماذج قديمة وحالة أحرف مختلطة. الأنبوب يوحّد الشكل قبل `HoldsService`. الخدمة تبحث عن `HLD-240` فقط. نموذج يرسل `HOLD-240` بلا أرقام كافية يُرفض بـ `400` ورسالة تذكر الشكل. السجل لا يمتلئ باستعلامات فاشلة.

## أخطاء شائعة

- نسخ `ParseIntPipe` المدمج في ملف جديد وتغيير الرسالة فقط. استخدم المدمج وخيار `exceptionFactory` إن كانت الرسالة هي الفرق.
- رمي `Error` عادي فيصبح `500`. ارمِ `HttpException` أو `BadRequestException` حتى يبقى الفشل خطأ عميل.
- الاعتماد على `metadata.metatype` لواجهة. لن تصلك.
- جعل الأنبوب عامًا عبر `useGlobalPipes` وهو يفترض نص قيد. سيعمل على كل المعاملات ويكسر الأجسام.

## متى نستخدمه؟

لقاعدة نطاق على قيمة واحدة أو على كائن صغير لا تعبّر عنه الأنابيب المدمجة، مع اختبار مباشر على `transform`.

## متى لا نستخدمه؟

حين يوجد اسم مدمج يطابق القاعدة، أو حين تكون القواعد زينة على عدة حقول. الثاني `ValidationPipe`. وحين يكون العمل جلب سجل. ذلك خدمة.

## مقارنة

| | مدمج | `HoldCodePipe` |
| --- | --- | --- |
| القاعدة | عامة: عدد، UUID، منطق | نمط المكتب `HLD-` |
| من يختبرها | اختبار الإطار | اختبار `transform` عندك |
| الربط | صنف أو نسخة بخيارات | صنف على `@Param` |

## تمارين

### تمرين سهل

ما الدالة الوحيدة التي يجب أن يوفّرها الصنف؟

<details>
<summary>الحل</summary>

`transform`. الواجهة `PipeTransform` تثبّتها.

</details>

### تمرين متوسط

كيف ترفض استخدام `HoldCodePipe` إن رُبط على `body` بالخطأ؟

<details>
<summary>الحل</summary>

اقرأ `metadata.type`. إن لم تكن `param` ارمِ `BadRequestException` برسالة أن الأنبوب لمسار القيد فقط.

</details>

### تمرين متقدم

البادئة `HLD` صارت تأتي من إعداد. كيف تحقنها من غير `new` في المتحكم؟

<details>
<summary>الحل</summary>

`constructor(private readonly prefix: HoldPrefix)` و `HoldPrefix` مزوّد في الوحدة. في المعامل مرّر الصنف `HoldCodePipe` لا نسخة، حتى تبنيه الحاوية. الربط العام خارج الوحدة بـ `new` لا يحقن.

</details>

## أسئلة مراجعة

- متى تكتب أنبوبًا مخصصًا؟
- ماذا تغيّر قيمة الرجوع في المعالج؟
- لماذا `BadRequestException` بدل `Error`؟
- كيف تختبر النمط بلا HTTP؟

## الخلاصة

الأنبوب المخصص صنف `transform` لقاعدة نطاق. يرجع قيمة مطبّعة أو يرمي خطأ عميل. ما يغطيه اسم مدمج يبقى على ذلك الاسم.

## ماذا بعد؟

الدرس التالي أماكن الربط الأربعة وترتيب تشغيلها، بما فيه معاملات المسار من الأخير إلى الأول.

## المصادر الرسمية

- [Pipes](https://docs.nestjs.com/pipes)
- [Request lifecycle](https://docs.nestjs.com/faq/request-lifecycle)
