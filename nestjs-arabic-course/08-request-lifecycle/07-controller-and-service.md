# Controller و Service

## الهدف من هذا الدرس

تحديد وسط الدورة: بعد Pipes تُستدعى دالة Controller، وهي التي تستدعي Service، والقيمة تعود إلى Interceptors الخروج.

## المشكلة التي يحلها

وضع القاعدة في الدالة يجعل وسط الرسم يحمل HTTP وقرار الإعارة معًا. وضع HTTP داخل Service يجعل الخدمة تنتظر Pipe و Guard. الوسط الرسمي ضيق: المعالج يستلم قيمًا جاهزة، ينادي الخدمة، يعيد الناتج أو يرمي.

## الفكرة ببساطة

البند 6 معالج الطريقة. البند 7 الخدمة إن وُجدت. اسم الدالة لا معنى له في المطابقة؛ المطابقة تمت قبل Middleware انتهت وقبل Guards. هنا التنفيذ فقط. نجاح الإرجاع يدخل طريق Interceptors من المسار إلى العام ثم طبقة الاستجابة التي عرفتها في Controllers: كائن يُسلسل JSON، و POST افتراضيًا 201.

## التعريف التقني

```ts
@Patch(':id')
update(@Param('id') id: string, @Body() body: UpdateLoanDto) {
  return this.loans.update(id, body);
}
```

`id` و `body` هما ناتج Pipes. `update` على الخدمة هي القاعدة. ما تعيده الدالة هو ما تصعد به سلسلة `Observable` إن بقي الأسلوب القياسي.

رمي من المعالج أو من الخدمة يتخطى بقية طريق النجاح. Interceptors التي دخلت قبل الدالة تستطيع قراءة الخطأ في `catchError`. إن بقي غير ممسوك تذهب الدورة إلى Exception Filters من المسار إلى العام.

`try/catch` يبتلع الرمي يعيد قيمة نجاح. المرشح لا يراها. Interceptor الخروج يرى القيمة البديلة كأي إرجاع.

`@Res()` من غير `passthrough` يعني أنك تكتب الرد، فطريق التسلسل و Interceptors القياسي على القيمة المعادة لا يجري كالمعتاد. ذلك استثناء أسلوب في درس لاحق من هذا الفصل. الوسط الطبيعي `return`.

لا توجد مرحلة اسمها «بين Controller و Service» غير نداء الدالة. الخدمة مزود جاهز منذ الإقلاع في النطاق الافتراضي، لا كائنًا يُبنى بعد Pipe.

## Mental Model

```text
Pipes سلّمت الوسائط
  -> update في Controller
  -> LoansService.update
  -> قيمة
  -> Interceptors الخروج
```

أو رمي من أحد السهمين فيخرج عن طريق النجاح.

## أبسط مثال

```ts
import { Body, Controller, Param, Patch } from '@nestjs/common';
import { LoansService } from './loans.service.js';
import { UpdateLoanDto } from './update-loan.dto.js';

@Controller('loans')
export class LoansController {
  constructor(private readonly loans: LoansService) {}

  @Patch(':id')
  update(@Param('id') id: string, @Body() body: UpdateLoanDto) {
    return this.loans.update(id, body);
  }
}
```

## شرح المثال

- `LoansService` حُقنت عند بناء Controller، قبل أي طلب، ما دام Scope الافتراضي.
- `PATCH` ناجح يعيد كائن الخدمة برمز 200 ما لم يضبط `@HttpCode`.
- المعالج لا يقرأ `req` الخام ولا يفحص صلاحية؛ تلك مراحل سابقة.
- إن كانت `update` غير متزامنة، Nest تنتظر الوعد ثم تمرر الناتج إلى طريق الخروج.

## ماذا يحدث داخليًا؟

استدعاء الطريقة هو أول كود تطبيق يعمل بعد التحويل. أي Interceptor ما زال ينتظر `Observable` الذي يصدر عندما ترجع الدالة. الخدمة تعمل على نفس مكدس النداء: هي دالة عادية على النسخة المحقونة. وقتها جزء من زمن المعالج، لا مرحلة بإعادة ترتيب.

حقل على Singleton يتغير هنا مرئي للطلب التالي. ذلك أثر العمر لا أثر ترتيب الدورة. قيمة هذا الطلب تبقى في الوسائط `id` و `body` وفي الكائن المعاد.

انتهاء الدالة بقيمة يبدأ البند 8. انتهاءها برمي يبحث عن ممسك في أغلفة Interceptor ثم في المرشحات.

## مثال عملي

```ts
import { Injectable } from '@nestjs/common';
import { UpdateLoanDto } from './update-loan.dto.js';

@Injectable()
export class LoansService {
  update(id: string, body: UpdateLoanDto) {
    if (body.days < 1) {
      throw new Error('days');
    }
    return { id, days: body.days };
  }
}
```

`days` الأقل من 1 لا يرجع `{ id, days }` إلى Interceptor الخروج كنجاح. الخطأ يصعد. استثناء HTTP أوضح سيأتي مع Exception Filters؛ الرمي هنا يكفي لبيان أن الخدمة قادرة على إيقاف طريق النجاح. فحص `days < 1` قاعدة مجال. شكل الجسم الفارغ كان يمكن أن ترفضه Pipe قبل الدخول، فيبقى هذا الفحص لما تعرفه الخدمة عن الإعارة لا عن JSON.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| فتح اتصال أو فحص صلاحية داخل `update` في Controller | وسط الدورة صار يعيد مراحل سابقة |
| توقع بناء `LoansService` بعد Pipe لكل طلب | النسخة الافتراضية جاهزة قبل الطلب |
| `try/catch` يعيد `{ ok: false }` وتوقع مرشح | طريق النجاح يكمل بالقيمة |
| `@Res()` بلا `passthrough` مع `return` | التسلسل القياسي لا يرسل تلك القيمة |

## متى نستخدمه؟

لتنفيذ العملية بعد أن نجحت المحطات السابقة: قراءة وسيط محوّل، نداء Service، إرجاع القيمة. اجعل الدالة رفيعة حتى يبقى الوسط واضحًا في التتبع.

## متى لا نستخدمه؟

لترتيب Middleware أو لمنع Guard أو لتحويل Pipe. تلك تمت. ولا لتشكيل خطأ HTTP النهائي إن كان لديك مرشح لاحقًا؛ ارمِ ودع المرشح يرد، إلا عندما تريد إمساك الحالة داخل الخدمة عمدًا.

## مقارنة

| البند | العمل |
| --- | --- |
| 6 Controller | استلام وسائط جاهزة وإرجاع قيمة أو رمي |
| 7 Service | القاعدة على النسخة المحقونة |
| ما قبل 6 | اختيار مسار ومنع وتحويل |
| ما بعد 7 عند النجاح | Interceptors من المسار إلى العام ثم الاستجابة |

## تمارين

### تمرين سهل

هل تُبنى `LoansService` الافتراضية داخل `update` لكل `PATCH`؟

<details><summary>الحل</summary>

لا. الحاوية بنتها مع الإقلاع وحقنتها. `update` تستدعي الدالة على تلك النسخة.

</details>

### تمرين متوسط

`loans.update` ترمي. ما الذي لا يكمل في طريق النجاح؟

<details><summary>الحل</summary>

صعود قيمة نجاح عبر Interceptors الخروج إلى تسلسل 200. الخطأ يصعد في الأغلفة، فإن بقي غير ممسوك وصل إلى Exception Filters.

</details>

### تمرين متقدم

لماذا فحص «الأيام أقل من 1» قد يعيش في الخدمة بينما فحص «الجسم ليس JSON» لا يعيش فيها؟

<details><summary>الحل</summary>

JSON التالف يتوقف عند المحلل أو عند Pipe قبل البند 6. الأيام قاعدة إعارة بعد أن وصلت الأرقام محوّلة. وضع فحص JSON في الخدمة يعيد عمل مرحلة سابقة بعد أن فات موضعها.

</details>

## أسئلة مراجعة

- ماذا يستلم المعالج من المرحلة السابقة؟
- أين تذهب القيمة المعادة في طريق النجاح؟
- ماذا يفعل `try/catch` بمرشح الاستثناء؟

## الخلاصة

وسط الدورة دالة Controller تستدعي Service على نسخة جاهزة. الإرجاع يصعد إلى Interceptors الخروج. الرمي يوقف طريق النجاح ما دام غير ممسوك.

## ماذا بعد؟

انعكاس الأغلفة: Interceptors بعد Controller.

## المصادر الرسمية

- NestJS — Request lifecycle: https://docs.nestjs.com/faq/request-lifecycle
- NestJS — Controllers: https://docs.nestjs.com/controllers
