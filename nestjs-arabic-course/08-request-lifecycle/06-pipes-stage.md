# مرحلة Pipes

## الهدف من هذا الدرس

تثبيت متى تعمل Pipes: من العام إلى Controller إلى المسار ثم إلى معامل المسار، والمعاملات من الأخير إلى الأول.

## المشكلة التي يحلها

`@Body()` ثم `@Param()` ثم `@Query()` يوحي أن الجسم يُفحص أولًا. التوثيق يعكس هذا داخل المستوى: آخر معامل يُعالَج أولًا. و Pipe على الصنف تكمل معاملات الدالة قبل أن تبدأ Pipe على `@Patch`. خلط المستويين يجعل التحقق العام يبدو كأنه داخل كل معامل مع تحقق المسار، وهو غير ترتيب Nest.

## الفكرة ببساطة

بعد دخول Interceptors وقبل الدالة: Pipes العامة، ثم Pipes الـ Controller، ثم Pipes المسار، ثم Pipes الخاصة بالمعامل. `@UsePipes(A, B)` يشغّل `A` ثم `B` على ذلك المستوى. كل مستوى يمشي المعاملات من الأخير إلى الأول قبل الانتقال إلى المستوى التالي.

`ValidationPipe` و `ParseIntPipe` أداتا تحويل وتحقق. إتقانهما فصل Pipes. هنا موضعهما في الدورة فقط.

## التعريف التقني

المثال المكافئ لتوثيق Nest، على إعارة:

```ts
@UsePipes(GeneralValidationPipe)
@Controller('loans')
export class LoansController {
  @UsePipes(RouteSpecificPipe)
  @Patch(':id')
  update(
    @Body() body: UpdateLoanDto,
    @Param() params: LoanParams,
    @Query() query: LoanQuery,
  ) {
    return this.loans.update(body, params, query);
  }
}
```

`GeneralValidationPipe` تعمل على `query` ثم `params` ثم `body`. بعد أن تنتهي هذه الجولة تبدأ `RouteSpecificPipe` الجولة نفسها: `query` ثم `params` ثم `body`. أي Pipe على معامل بعينه تعمل بعد مستويي Controller والمسار، وما زال آخر معامل أولًا.

Pipe عامة تسبق `GeneralValidationPipe` وتمشي المعاملات من الأخير إلى الأول أيضًا.

المعالج لا يبدأ إلا بعد هذه الجولات. رمي Pipe يتخطى الدالة والخدمة. Interceptor الذي لف الدخول يقدر أن يرى الخطأ في `catchError`. إن بقي غير ممسوك تنتقل Exception Filters.

## Mental Model

```text
لكل مستوى من: عام -> Controller -> مسار
  آخر معامل -> … -> أول معامل
ثم Pipes المكتوبة على المعاملات نفسها
  من الأخير إلى الأول
ثم جسم الدالة
```

## أبسط مثال

```ts
import { Body, Controller, Param, Patch, Query, UsePipes } from '@nestjs/common';
import { GeneralValidationPipe } from './general-validation.pipe.js';
import { LoanQuery } from './loan.query.js';
import { LoanParams } from './loan.params.js';
import { LoansService } from './loans.service.js';
import { RouteSpecificPipe } from './route-specific.pipe.js';
import { UpdateLoanDto } from './update-loan.dto.js';

@UsePipes(GeneralValidationPipe)
@Controller('loans')
export class LoansController {
  constructor(private readonly loans: LoansService) {}

  @UsePipes(RouteSpecificPipe)
  @Patch(':id')
  update(
    @Body() body: UpdateLoanDto,
    @Param() params: LoanParams,
    @Query() query: LoanQuery,
  ) {
    return this.loans.update(body, params, query);
  }
}
```

## شرح المثال

- ترتيب المعاملات في التوقيع: جسم، ثم مسار، ثم استعلام. المعالجة داخل المستوى تبدأ من الاستعلام.
- `RouteSpecificPipe` لا تبدأ على `query` قبل أن تكمل `GeneralValidationPipe` الجسم.
- `update` في الخدمة تستلم القيم بعد الجولتين.
- الأصناف هنا أسماء مواضع. جسم `transform` يُكتب في فصل Pipes.

## ماذا يحدث داخليًا؟

Nest تجمع Pipes كل مستوى وتنفذها أولًا فأولًا على المستوى، ولكل Pipe تمشي قائمة المعاملات عكس كتابتها. لذلك آخر معامل في المصدر هو أول من يمر على `GeneralValidationPipe`.

Pipe على المعامل، مثل `ParseIntPipe` داخل `@Param('id', ParseIntPipe)`، تُجدول بعد `@UsePipes` الخاصة بالمسار و Controller والعام. إن وُجدت على أكثر من معامل، فالأخير منها يعمل قبل الأول.

نجاح الجميع يستدعي الدالة بالقيم الناتجة. النوع `number` بعد `ParseIntPipe` قيمة محوّلة، لا تعليق TypeScript. قبل هذه المرحلة القيمة كانت نص المسار كما في درس Route Params.

## مثال عملي

```ts
@Patch(':id')
@UsePipes(RouteSpecificPipe)
extend(
  @Body() body: UpdateLoanDto,
  @Param('id', ParseIntPipe) id: number,
) {
  return this.loans.extend(id, body);
}
```

افترض `GeneralValidationPipe` ما زالت على الصنف. الترتيب: الجولة العامة على `id` ثم على `body`، ثم `RouteSpecificPipe` على `id` ثم على `body`، ثم `ParseIntPipe` على `id` لأنه Pipe معامل والـ `body` بلا Pipe معامل في هذا التوقيع. الدالة ترى `id` عددًا بعد ذلك. فشل `ParseIntPipe` لا ينادي `extend`.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| افتراض أن `@Body` يُفحص قبل `@Query` لأنهما كُتبا هكذا | داخل المستوى الأخير أولًا |
| افتراض أن Pipe المسار تتداخل مع Pipe الصنف على المعامل نفسه قبل إكمال الصنف | الصنف يكمل الجولة ثم يبدأ المسار |
| الاعتماد على تعليق `number` من غير Pipe | النص يبقى نصًا وتصل الخدمة كما في فصل Controllers |
| وضع Pipe قبل Guard في توقعك | Guards مرحلة أسبق؛ المنع يلغي هذه المرحلة |

## متى نستخدمه؟

لتحويل معامل أو رفض شكل البيانات قبل أن تلمس Service القيمة: عدد في المسار، أو جسم يطابق DTO. اربط العام لما يشمل التطبيق، والمسار لما يخص دالة، والمعامل لما يخص وسيطًا واحدًا.

## متى لا نستخدمه؟

لمنع عضو. ذلك Guard ويحدث قبل هذه المرحلة، فلن تُستهلك Pipe على طلب مرفوض. لختم الطلب قبل اختيار المسار. ذلك Middleware. لتشكيل الاستجابة بعد الدالة. ذلك Interceptor على طريق الخروج.

## مقارنة

| الربط | متى بالنسبة لغيره |
| --- | --- |
| Pipe عامة | أول جولة معاملات، من الأخير إلى الأول |
| `@UsePipes` على الصنف | بعد العام، وتكمل جولتها قبل المسار |
| `@UsePipes` على الدالة | بعد الصنف، بالجولة نفسها |
| Pipe على المعامل | بعد المستويات الثلاثة، وآخر معامل أولًا |
| داخل `@UsePipes(A, B)` | `A` ثم `B` |

## تمارين

### تمرين سهل

في المثال، ماذا تعالج `GeneralValidationPipe` أولًا: `body` أم `query`؟

<details><summary>الحل</summary>

`query`، لأنه آخر معامل. ثم `params` ثم `body`.

</details>

### تمرين متوسط

متى تبدأ `RouteSpecificPipe` على `query`؟

<details><summary>الحل</summary>

بعد أن تنهي `GeneralValidationPipe` جولتها كلها، بما فيها `body`. ثم تمشي `RouteSpecificPipe` من `query` إلى `body` أيضًا.

</details>

### تمرين متقدم

أين يقع `ParseIntPipe` على `@Param` بالنسبة إلى `@UsePipes` على الصنف وعلى الدالة؟

<details><summary>الحل</summary>

بعدهما. Pipes المعامل آخر بند في المرحلة، ومن آخر معامل إلى أول معامل إن تعددت.

</details>

## أسئلة مراجعة

- ما ترتيب مستويات Pipes الأربعة؟
- من أي معامل تبدأ جولة المستوى؟
- ماذا يحدث للدالة إن رمت Pipe؟

## الخلاصة

Pipes بعد دخول Interceptors وقبل الدالة: عام، ثم Controller، ثم مسار، ثم معامل. كل مستوى يكمل المعاملات من الأخير إلى الأول قبل المستوى التالي، و `@UsePipes` داخل المستوى من الأول المكتوب إلى التالي.

## ماذا بعد؟

بعد نجاح التحويل: Controller و Service.

## المصادر الرسمية

- NestJS — Request lifecycle (Pipes): https://docs.nestjs.com/faq/request-lifecycle
