# Built-in HTTP Exceptions

## الهدف من هذا الدرس

اختيار الفئة الجاهزة التي تطابق رمز الحالة بدل بناء `HttpException` عام في كل مرة.

## المشكلة التي يحلها

الفريق يكتب `new HttpException('...', 404)` و `new HttpException('...', 403)` فتتفاوت الرسائل ويصعب التقاط نوع واحد في فلتر. الفئات تثبت الرمز في الاسم.

## الفكرة ببساطة

خمس فئات من `@nestjs/common` تغطي رفض الشكل، وغياب الهوية، ونقص الإذن، وغياب المورد، والتعارض:

| الصنف | الرمز |
| --- | --- |
| `BadRequestException` | `400` |
| `UnauthorizedException` | `401` |
| `ForbiddenException` | `403` |
| `NotFoundException` | `404` |
| `ConflictException` | `409` |

كلها تمتد من `HttpException`. ترميها وتترك الفلتر المدمج يكتب JSON.

## التعريف التقني

كل فئة تقبل الرسالة كنص أو كائنًا لجسم مخصص، مثل الأب. من غير وسيط تستخدم رسالة افتراضية مرتبطة بالرمز.

`BadRequestException` هو ما ترميه `ValidationPipe` و `ParseIntPipe` عند الفشل. لا تعد رميه في الخدمة لخطأ زينة. الأنبوب فعل ذلك. ارمه من الخدمة حين يكون الطلب مفهومًا شكلاً لكن قاعدة خفيفة لا تحتاج مخزنًا ما زالت رفض شكل، بحذر حتى لا تخلط الأعمال بالشكل.

`UnauthorizedException` للهوية الناقصة، كما في الحارس التعليمي. `ForbiddenException` هو ما يرميه الإطار حين ترجع `canActivate` القيمة `false`، ويمكنك رميه بنفسك برسالة أوضح.

`NotFoundException` لغياب تذكرة. `ConflictException` لمقعد مبيع أو لتذكرة أُلغيت مرتين.

فئات أخرى موجودة في الحزمة، مثل `RequestTimeoutException` الذي مر في المهلة. هذا الدرس يثبت الخمسة التي يسميها عنوان المسار.

التقاط النوع في فلتر لاحقًا يكون `@Catch(NotFoundException)` لا التقاط الأب إن أردت شكلًا لمورد غائب فقط.

## Mental Model

```text
شكل تالف          400 BadRequest
لا هوية           401 Unauthorized
هوية بلا إذن      403 Forbidden
لا تذكرة          404 NotFound
مقعد مبيع         409 Conflict
عطل غير معروف     500 من غير هذه الفئات
```

## أبسط مثال

```ts
import {
  ConflictException,
  Injectable,
  NotFoundException,
} from '@nestjs/common';

@Injectable()
export class TicketsService {
  private readonly sold = new Set<string>(['s1:4']);

  reserve(showId: string, seat: number): { showId: string; seat: number } {
    const key = `${showId}:${seat}`;
    if (this.sold.has(key)) {
      throw new ConflictException(`Seat ${seat} is already sold`);
    }
    this.sold.add(key);
    return { showId, seat };
  }

  find(showId: string, seat: number): { showId: string; seat: number } {
    const key = `${showId}:${seat}`;
    if (!this.sold.has(key)) {
      throw new NotFoundException(`Ticket ${key} was not found`);
    }
    return { showId, seat };
  }
}
```

المثال ذاكرة في العملية حتى ترى الرمز. المخزن الحقيقي لاحقًا. الرمي نفسه يبقى.

## شرح المثال

- التعارض بعد أن الشكل صح: المقعد عدد، والعرض نص. `409` أعمال.
- الغياب `404` لا `500`.
- لا `try/catch` في المتحكم. Nest يكتب الجسم.
- `BadRequestException` غير مستخدم هنا لأن الأنبوب يملك فحص الشكل قبل `reserve`.

## ماذا يحدث داخليًا؟

كل فئة تضبط `status` في بناء الأب. الفلتر المدمج يقرأ `getStatus()` و `getResponse()`. الرسالة النصية تصبح `message`. العميل يفرّق الرموز من غير قراءة الجملة.

حارس يرجع `false` ينتج `ForbiddenException` من الإطار، فتظهر الرسالة الافتراضية `Forbidden resource` لا جملتك. إن أردت جملة الشباك فارمِ `new ForbiddenException('Desk cannot refund')` بدل `false`.

`ValidationPipe` لا يرمي `ConflictException`. لا توحّد الرمزين في الاختبار: `400` شكل، `409` حالة مقعد.

## مثال عملي

`POST /tickets` بمقعد مبيع يرجع `409`. `GET /tickets/s1/4` قبل البيع يرجع `404`. طلب `seat=0` يرجع `400` من الزينة إن كان `@Min(1)` قبل الخدمة، والدالة `reserve` لا تُستدعى. اختبارات ثلاثة، ثلاثة أصناف، ولا `Error` عام.

## أخطاء شائعة

- `NotFoundException` لمقعد مبيع. العميل يعيد المحاولة كأنه مسار خاطئ.
- `ConflictException` لبريد غير صالح.
- `ForbiddenException` عند غياب الترويسة. ذلك `401`.
- التقاط الخمسة في `catch` وإرجاع `200` مع حقل `error`.

## متى نستخدمه؟

كلما طابق المعنى أحد الرموز الخمسة. ارمِ الفئة واترك الطبقة تكتب الرد.

## متى لا نستخدمه؟

لعطل برمجة أو فشل شبكة غير متوقع قد يبقى `500` حتى لا تسمّي العطل «تعارضًا». ولجسم بمفاتيح غير الرسالة الافتراضية إن كان `HttpException` بالكائن أوضح، أو فئة خاصة في الدرس التالي.

## مقارنة

| الحالة في الشباك | الصنف |
| --- | --- |
| `seat` نص | `BadRequestException` عبر الأنبوب |
| لا `x-desk-name` | `UnauthorizedException` |
| شباك لا يسترد | `ForbiddenException` |
| لا صف تذكرة | `NotFoundException` |
| المقعد مبيع | `ConflictException` |

## تمارين

### تمرين سهل

أي صنف يرميه الإطار حين ترجع البوابة `false`؟

<details>
<summary>الحل</summary>

`ForbiddenException` برمز `403`.

</details>

### تمرين متوسط

مقعد مبيع وطلب بمعرف غير موجود. أي رمزين، ولماذا لا يتبادلان؟

<details>
<summary>الحل</summary>

المبيع `409` لأنه موجود وحالته تمنع الحجز. غير الموجود `404`. التبادل يجعل العميل يعيد إنشاء مسار خاطئ أو يظن التعارض غيابًا.

</details>

### تمرين متقدم

الخدمة ترمي `BadRequestException` لنقص المخزون. ما الاعتراض على الرمز، وما الفئة الأقرب؟

<details>
<summary>الحل</summary>

الشكل سليم والمخزون حالة أعمال. `ConflictException` أو فئة أعمال لاحقة أقرب من `400` الذي يوحي بأن JSON تالف.

</details>

## أسئلة مراجعة

- ما رموز الخمسة؟
- من يرمي `400` غالبًا قبل الخدمة؟
- متى تختار `401` لا `403`؟
- لماذا لا تُستخدم `NotFoundException` للمقعد المبيع؟

## الخلاصة

الفئة الجاهزة تثبت الرمز في الاسم. الخمسة تغطي الشكل والهوية والإذن والغياب والتعارض في الشباك. `Error` العاري يبقى للعطل غير المصنّف لا لهذه الحالات.

## ماذا بعد؟

الدرس التالي صنف يمتد من `HttpException` حين تحتاج اسمًا لجسم خطأ خاص بالتذاكر.

## المصادر الرسمية

- [Exception filters](https://docs.nestjs.com/exception-filters)
- [Built-in HTTP exceptions](https://docs.nestjs.com/exception-filters#built-in-http-exceptions)
