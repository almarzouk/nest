# Interceptors قبل Controller

## الهدف من هذا الدرس

تثبيت اتجاه الدخول إلى Interceptors: العام، ثم Controller، ثم المسار، ولماذا طريق العودة سينعكس.

## المشكلة التي يحلها

مؤقت يلف كل الطلبات، وآخر يلف إعارات Controller، وثالث يلف `PATCH` واحد. إن عمل مسار `PATCH` قبل المؤقت العام، فزمن العام لا يشمل ما قبله. Nest تدخل من الخارج إلى الدالة، كـ Guards، ثم تعكس الخروج لأن العائد `Observable`.

## الفكرة ببساطة

Interceptor يلف ما بعده. قبل المعالج الترتيب: عام، Controller، مسار. `next.handle()` يعيد `Observable`. من دخل أولًا يكون آخر من يرى القيمة في طريق العودة. درس «بعد Controller» يفصل الخروج. هنا نثبت الدخول وسبب الانعكاس.

الحد الأدنى من RxJS وكتابة الصنف في فصل Interceptors. `catchError` مذكور لأن التوثيق يربطه بهذه المرحلة: أخطاء Pipes و Controller و Service يمكن قراءتها فيه.

## التعريف التقني

المستويات تطابق Guards. `@UseInterceptors(TimingInterceptor, AuditInterceptor)` على الصنف يشغّل `TimingInterceptor` ثم `AuditInterceptor` في الدخول، بعد أي Interceptor عام، وقبل زينة الدالة.

الانعكاس لأن كل Interceptor يعيد `Observable` يغلّف نداء `next.handle()`. السلسلة تُحسم أول داخل آخر خارج: أول `Observable` اشتُرك فيه هو آخر من يصدر القيمة النهائية.

```text
دخول: Global -> Timing -> Audit -> RouteInterceptor -> … المعالج
خروج: RouteInterceptor -> Audit -> Timing -> Global
```

هذا الانعكاس خاص بـ Interceptors. Guards لا تملك طريق عودة بالقيمة.

خطأ من Pipe أو من المعالج أو من Service يصعد في هذه الـ `Observable`. `catchError` يستطيع قراءته. إن أعاد قيمة بديلة أصبح الخطأ ممسوكًا. إن أعاد رميه بقي غير ممسوك وانتقل إلى Exception Filters.

## Mental Model

```text
غلاف عام
  غلاف Controller
    غلاف المسار
      Pipes ثم الدالة
    يرى القيمة أولًا في الخروج
  ثم غلاف Controller
ثم الغلاف العام
```

## أبسط مثال

```ts
import { Controller, Patch, UseInterceptors } from '@nestjs/common';
import { AuditInterceptor } from './audit.interceptor.js';
import { LoansService } from './loans.service.js';
import { RouteInterceptor } from './route.interceptor.js';
import { TimingInterceptor } from './timing.interceptor.js';

@UseInterceptors(TimingInterceptor, AuditInterceptor)
@Controller('loans')
export class LoansController {
  constructor(private readonly loans: LoansService) {}

  @UseInterceptors(RouteInterceptor)
  @Patch(':id')
  update() {
    return this.loans.touch();
  }
}
```

## شرح المثال

- افترض `APP_INTERCEPTOR` أو `useGlobalInterceptors` باسم `GlobalTiming` في التطبيق. دخوله قبل `TimingInterceptor`.
- الدخول إلى `update`: العام، ثم `TimingInterceptor`، ثم `AuditInterceptor`، ثم `RouteInterceptor`، ثم Pipes، ثم الدالة.
- الخروج يعكس هذه الأسماء. جدول الخروج درسه المستقل حتى لا يختلط مع أرقام الدخول.
- `touch` ما زالت قاعدة الخدمة. المؤقت لا يفتح إعارة.

## ماذا يحدث داخليًا؟

Nest تبني سلسلة الاشتراك عند تنفيذ المسار: كل Interceptor يستدعي التالي عبر `next.handle()` ويغلّف الناتج. الاشتراك يبدأ من الخارج، والتنفيذ الفعلي للمعالج يحدث عندما تصل السلسلة إلى النهاية، أي بعد Pipes. لذلك Interceptor الدخول يحيط بـ Pipes حتى لو ظهرت Pipes بعده في القائمة المرقمة: اللف يبدأ قبلها، والدالة تعمل بعد Pipes، والقيمة تصعد في الأغلفة.

هذا لا يغيّر رقم Pipes في التسلسل الرسمي: Pipes بعد بدء الدخول وقبل الدالة. اللف موجود حولها لأنه `Observable` واحد.

`catchError` على الغلاف الخارجي يرى خطأ الغلاف الداخلي إن لم يُمسك هناك. هذا غير مرشح Exception Filter: المرشح بعد أن يبقى الخطأ غير ممسوك.

## مثال عملي

قياس زمن `PATCH /loans/9` شاملاً Pipe تحويل `id` ونداء الخدمة: المؤقت العام أو `TimingInterceptor` يجب أن يكون خارج Pipe في اللف، وهو كذلك عندما يُربط عامًا أو على الصنف. مؤقت على المسار فقط لا يشمل وقت Interceptor الصنف في قياس «ما قبل الدالة» إلا كغلاف داخلي. ضع قياس العملية كلها على المستوى الأعم حتى يحيط بالباقي في الدخول ويخرج آخرًا فيرى الزمن الكامل.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| توقع خروج Interceptor بترتيب الدخول نفسه | أول داخل آخر خارج |
| وضع قياس كلي على زينة الدالة فقط | الأغلفة الأعم خارجه في الدخول وداخله في القياس العكسي |
| ابتلاع الخطأ في `catchError` ثم توقع المرشح | الخطأ صار ممسوكًا |
| كتابة منطق منع داخل Interceptor بدل Guard | المنع المتأخر يأتي بعد حراس أخرى وقد بعد بدء اللف |

## متى نستخدمه؟

عندما تحتاج لفًا يرى ما قبل الدالة وما بعدها: زمن، تتبع، أو تحويل القيمة اللاحقة. اربطه على أوسع مستوى يجب أن يحيط بالباقي.

## متى لا نستخدمه؟

للمنع. ذلك Guard قبل هذه المرحلة. لتحويل معامل. ذلك Pipe داخل اللف وقبل الدالة. لختم الطلب قبل اختيار المسار. ذلك Middleware. عمق `pipe` و `switchMap` مؤجل إلى فصل Interceptors.

## مقارنة

| | Guards | Interceptors الدخول |
| --- | --- | --- |
| الترتيب | عام، Controller، مسار | الاتجاه نفسه |
| طريق العودة | لا قيمة ملفوفة | تنعكس لأن العائد `Observable` |
| الفشل | يوقف قبل اللف | `catchError` قد يمسك ما بعد الدخول |
| `false` | منع | ليس نموذج Guard |

## تمارين

### تمرين سهل

ما ترتيب الدخول لـ Interceptor عام وآخر على الصنف وآخر على الدالة؟

<details><summary>الحل</summary>

العام، ثم الصنف، ثم الدالة.

</details>

### تمرين متوسط

لماذا يخرج `TimingInterceptor` على الصنف بعد `RouteInterceptor` في طريق القيمة؟

<details><summary>الحل</summary>

`RouteInterceptor` دخل لاحقًا فغلافه داخلي. `Observable` يُحسم من الداخل إلى الخارج، فالغلاف الداخلي يصدر قبل غلاف الصنف، والعام آخرًا.

</details>

### تمرين متقدم

Interceptor عام أعاد قيمة من `catchError` بعد رمي Pipe. هل يعمل Exception Filter المسار؟

<details><summary>الحل</summary>

لا، إن أصبحت القيمة بديلاً عن الخطأ. المرشح يرى غير الممسوك. إعادة الرمي من `catchError` تبقي الاستثناء فيصل إلى المرشحات.

</details>

## أسئلة مراجعة

- ما اتجاه الدخول؟
- لماذا ينعكس الخروج؟
- أي أخطاء يذكر التوثيق أن `catchError` يقدر أن يقرأها؟

## الخلاصة

دخول Interceptors من العام إلى Controller إلى المسار، وهي تلف ما بعدها بما فيه Pipes والدالة. الخروج يعكس الترتيب لأن `next.handle()` يعيد `Observable` يُحسم أول داخل آخر خارج.

## ماذا بعد؟

تحويل المعاملات قبل الدالة: مرحلة Pipes.

## المصادر الرسمية

- NestJS — Request lifecycle (Interceptors): https://docs.nestjs.com/faq/request-lifecycle
