# ValidationPipe

## الهدف من هذا الدرس

تشغيل `ValidationPipe` المدمج على صنف مزيّن، ومعرفة متى يتخطى المعامل.

## المشكلة التي يحلها

زينة `class-validator` على `ReserveSeatDto` لا تعمل حتى يستدعي شيء `validate`. كتابة الأنبوب التعليمي من صفحة Pipes تعيد ما يوفره الإطار بغنى أقل.

## الفكرة ببساطة

`new ValidationPipe()` عامًا على التطبيق، والمعامل نوعه صنف عليه زينة. الفشل `400` و `message` مصفوفة رسائل. النجاح يمرر القيمة إلى الدالة. الخيارات التي تحذف حقولًا أو تحوّل النسخة تأتي بعد هذا الدرس.

## التعريف التقني

الأنبوب من `@nestjs/common`. الحزمتان `class-validator` و `class-transformer`.

```ts
app.useGlobalPipes(new ValidationPipe());
```

الربط على الدالة `@UsePipes(new ValidationPipe())` أو على المعامل `@Body(new ValidationPipe())` حين لا تريد عمومية بعد.

خيارات البناء التي تذكرها الواجهة في الصفحة تشمل `transform` و `disableErrorMessages` و `exceptionFactory` و `errorFormat`، وترث خيارات `class-validator` مثل `whitelist` و `forbidNonWhitelisted`. هذا الدرس يتركها على افتراضاتها.

الأنبوب يفحص معاملًا نوعه صنف. يتخطى `String` و `Number` و `Boolean` و `Array` و `Object`. لذلك `@Param('id') id: string` لا يُفحص هنا. استخدم أنبوب تحليل أو صنفًا مثل `FindSeatParams` عليه `@IsNumberString()` إن أردت فحص كائن المعاملات كله.

لا `import type` للصنف. لا واجهة مكان الصنف. لا نوع عام عارٍ لعناصر مصفوفة.

عند بريد غير صالح، الصفحة تعرض شكلًا مثل:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

الرمز الافتراضي يمكن تغييره بـ `errorHttpStatusCode` ضمن خيارات المكتبة. الافتراضي `BadRequestException`.

## Mental Model

```text
metatype صنف مزيّن؟  نعم → plainToInstance ثم validate → 400 أو مرور
metatype نوع أصلي؟    نعم → تمرير القيمة كما هي
```

## أبسط مثال

```ts
import { IsInt, IsString, Min } from 'class-validator';

export class ReserveSeatDto {
  @IsString()
  showId: string;

  @IsInt()
  @Min(1)
  seat: number;
}
```

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

const app = await NestFactory.create(AppModule);
app.useGlobalPipes(new ValidationPipe());
await app.listen(3000);
```

```ts
@Post()
reserve(@Body() dto: ReserveSeatDto): ReserveSeatDto {
  return dto;
}
```

`{ "showId": "s1", "seat": 0 }` يرجع `400`. `{ "showId": "s1", "seat": 4 }` يدخل `reserve`.

## شرح المثال

- الربط مرة في الإقلاع يغطي كل معامل صنف.
- الدالة ترجع `dto` حتى ترى ماذا مر. الخدمة في التطبيق الحقيقي تُستدعى هنا.
- حقل `note` زائد في JSON يمر مع هذا البناء لأن `whitelist` لم يُفتح.
- المعالج يستلم كائنًا قد لا يكون `instanceof ReserveSeatDto` إلى أن تفعّل `transform`. الفحص تم على نسخة داخلية.

## ماذا يحدث داخليًا؟

الإطار يمرر `ArgumentMetadata.metatype`. الأنبوب إن وجد صنفًا خارج قائمة الأنواع الأصلية يستدعي `plainToInstance` ثم `validate`. أخطاء غير فارغة تذهب إلى `exceptionFactory` الافتراضي الذي يرمي `BadRequestException` برسائل القيود.

معامل ثانٍ على الدالة نفسها من نوع أصلي يُمرَّر. أنبوب عام لا يعني أن كل المعاملات تُفحص بالقواعد نفسها.

نسخة `new ValidationPipe()` في `main.ts` لا تُحقن. إن احتاج الأنبوب خدمة، فالربط `{ provide: APP_PIPE, useClass: ValidationPipe }` كما في صفحة الأنابيب. الخيارات عندها تُمرَّر بطريقة المزود المناسبة، والغالب في هذا الفصل نسخة الإقلاع لأن الخيارات قيم ثابتة.

فشل الأنبوب يسبق جسم `reserve`. فلتر المسار يراه.

## مثال عملي

`POST /reservations` على الصالة. العميل يرسل `seat` كنص `"4"` من نموذج. `@IsInt()` يرفض لأن القيمة ليست عددًا. ذلك صحيح مع الافتراض. تحويل النص إلى عدد ليس هذا الدرس. مسار `GET /shows/:id` يبقى على `ParseUUIDPipe` أو نص، ولا تنتظر أن الأنبوب العام يرفض `id` الفارغ إن كان النوع `string`.

## أخطاء شائعة

- نسيان `app.useGlobalPipes` ثم لوم الزينة.
- توقع رفض الحقل الزائد من هذا البناء.
- فحص مصفوفة `ReserveSeatDto[]` من غير `ParseArrayPipe` أو صنف يغلّف العناصر.
- بناء الأنبوب مرتين، مرة في `main.ts` ومرة بخيارات أضيق على المسار، من غير أن تعرف أن مستوى المسار يعمل ضمن ترتيب الأنابيب. الأضيق يجب أن يكون مقصودًا.

## متى نستخدمه؟

سياسة واحدة للتطبيق: كل جسم صنف يُفحص بزينة `class-validator` قبل المتحكم.

## متى لا نستخدمه؟

لمخطط Zod. ذلك `StandardSchemaValidationPipe`. ولعدد مسار واحد حيث `ParseIntPipe` أوضح. ولا تفتح كل الخيارات في السطر الأول قبل أن تعرف أثر كل واحد.

## مقارنة

| الربط | متى |
| --- | --- |
| `useGlobalPipes` | كل الأصناف في تطبيق HTTP |
| `@UsePipes` على متحكم | مجموعة مسارات فقط |
| `@Body(new ValidationPipe())` | جسم دالة واحدة |
| `APP_PIPE` | حين تحتاج حقنًا |

## تمارين

### تمرين سهل

ما رمز الحالة الافتراضي لفشل `@Min(1)` مع هذا الأنبوب؟

<details>
<summary>الحل</summary>

`400` عبر `BadRequestException`.

</details>

### تمرين متوسط

لماذا يمر `@Query('q') q: string` بلا رسالة فحص؟

<details>
<summary>الحل</summary>

`metatype` نوع أصلي `String`. الأنبوب يتخطاه. القاعدة إن لزم تكون على صنف استعلام أو على أنبوب تحليل.

</details>

### تمرين متقدم

تريد رسائل `class-validator` على جسم الحجز، وحقن مسجّل داخل أنبوب عام. أي ربط يفشل، وما البديل؟

<details>
<summary>الحل</summary>

`new ValidationPipe()` في `main.ts` لا يحقن. سجّل `APP_PIPE` في وحدة ترى المسجّل. الرسائل تبقى من المكتبة ما دامت الزينة على الصنف.

</details>

## أسئلة مراجعة

- أي حزم يحتاج الأنبوب؟
- متى يتخطى المعامل؟
- هل الافتراض يحذف الحقول الزائدة؟
- هل الدالة تُستدعى عند فشل `validate`؟

## الخلاصة

`ValidationPipe` العام يفحص أصناف العقد ويرفض بـ `400`. الأنواع الأصلية تمر. الحذف والتحويل وشكل الرسالة خيارات منفصلة تُضاف حين تعرف أثرها.

## ماذا بعد؟

الدرس التالي خيارا `whitelist` و `forbidNonWhitelisted`: إسقاط الحقل الزائد أو رفض الطلب بسببه.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
- [Pipes](https://docs.nestjs.com/pipes)
