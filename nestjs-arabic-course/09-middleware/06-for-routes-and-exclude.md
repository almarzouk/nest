# forRoutes و exclude

## الهدف من هذا الدرس

تحديد المسارات التي يجري عليها Middleware، واستثناء ما يجب أن يبقى خارج السلسلة.

## المشكلة التي يحلها

ختم التتبع على `GET /health` يلوّث فحص الحياة، وختم الإعارات على كل التطبيق يضيف ترويسة لا تعني شيئًا لمسار الأعضاء. التقييد جزء من التسجيل.

## الفكرة ببساطة

بعد `apply` تنادي `forRoutes` بما يجب تغطيته، و `exclude` بما يُطرح من تلك التغطية. `exclude` تأتي في السلسلة قبل `forRoutes`.

## التعريف التقني

`forRoutes` يقبل:

- نص مسار مثل `'loans'`.
- عدة نصوص.
- كائن `RouteInfo`: `{ path, method }` مع `RequestMethod`.
- صنف متحكم، أو عدة أصناف. هذا الشكل الذي تقوله الصفحة في أغلب الحالات.

`exclude` يقبل نصًا، أو عدة نصوص، أو كائنات `RouteInfo`. الاستثناء يُطرح من مجموعة `forRoutes`.

المسارات النمطية تستخدم معاملًا باسم، كما في الصفحة:

```ts
{ path: 'loans/*rest', method: RequestMethod.ALL }
```

الاسم `rest` مثال. أي اسم يصلح، مثل `*splat` في مثال التوثيق. النص `loans/*rest` يطابق `loans/1` و `loans/abc` ولا يطابق `loans` وحده. لجعل الذيل اختياريًا:

```ts
{ path: 'loans/{*rest}', method: RequestMethod.ALL }
```

تلميح الصفحة لربط صنف بكل المسارات من الوحدة يستخدم `forRoutes('*')`. النمط المسمّى أعلاه هو شكل الجزء من الشجرة.

`RequestMethod.GET` و `POST` و `ALL` تميّز الفعل. نص `'loans'` من غير كائن يشمل أفعال ذلك المسار حسب سلوك الربط بالنص.

## Mental Model

```text
apply(Trace)
  exclude  GET /health
  forRoutes LoansController
الناتج = مسارات المتحكم ناقص ما استثنيت
```

الاستثناء لا يسجّل Middleware آخر. هو ثقب في المجموعة.

## أبسط مثال

```ts
import {
  type MiddlewareConsumer,
  type NestModule,
  RequestMethod,
} from '@nestjs/common';
import { LoansController } from './loans.controller.js';
import { traceLoan } from './trace-loan.middleware.js';

export class LoansModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer
      .apply(traceLoan)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
        { path: 'loans', method: RequestMethod.POST },
      )
      .forRoutes(LoansController);
  }
}
```

`GET /loans` يمر على `traceLoan` إن كان داخل `LoansController`. `GET /health` و `POST /loans` لا يمران بهذه السلسلة، بشرط أن تلك المسارات أصلًا ضمن ما يغطيه `forRoutes`.

## شرح المثال

- `exclude` قبل `forRoutes` كما في مثال الصفحة. السلسلة: طبّق، اطرح، ثم ثبّت المجموعة.
- استثناء `POST /loans` يعني أن إنشاء الإعارة بلا ترويسة التتبع. افعل ذلك بقرار واضح.
- تمرير `LoansController` أوضح من سرد النصوص، لأن مسارًا جديدًا على المتحكم يدخل المجموعة وحدها.

الاستثناء لازم حين يكون المسار داخل المجموعة، مثل `forRoutes('*')`. مسار على متحكم لم يُمرَّر أصلًا لا يحتاج `exclude`.

## ماذا يحدث داخليًا؟

عند الإقلاع تتحول القيم إلى قواعد مطابقة. الصفحة تقول إن `exclude` يعتمد path-to-regexp. عند الطلب تُختبر القواعد قبل تشغيل `use` أو الدالة. عدم التطابق يعني أن هذه السلسلة تُتخطى. سلاسل أخرى، و `app.use`، قد تظل تعمل.

`forRoutes('loans')` يطابق مسارًا بهذا الاسم. صنف المتحكم يطابق ما أعلنه `@Controller` ودواله. إن انتقلت البادئة إلى `circulation/loans`، الربط بالصنف يتبعها. الربط بالنص `'loans'` قد يفوته.

النقطة `{*rest}` اختيارية. من غير الأقواس، المسار الأب وحده لا يطابق النمط ذا النجمة.

## مثال عملي

فحص `GET /health` يبقى رخيصًا وبلا ترويسات تتبع. الوحدة الجذر تربط `traceLoan` بـ `forRoutes('*')` وتستثني `{ path: 'health', method: RequestMethod.GET }`. مسار `GET /loans` يبقى مختومًا. هذا سبب `exclude` بجانب النجمة، لا بجانب متحكم لا يملك الصحة أصلًا.

## أخطاء شائعة

- استثناء مسار خارج `forRoutes` ثم الاعتقاد أنه يحمي مسار وحدة أخرى. الطرح من هذه السلسلة فقط.
- كتابة نجمة بلا اسم داخل نمط جزئي. الصفحة الحالية تستخدم اسمًا: `*splat` أو `{*splat}`.
- نسيان `RequestMethod` حين تريد فعلًا واحدًا.
- ربط نص `'loans'` بينما المتحكم معلّق على `'circulation/loans'`.

## متى نستخدمه؟

`forRoutes` في كل ربط. `exclude` حين تكون المجموعة أوسع من المسارات المرغوبة: نجمة، أو متحكم فيه مسار صحة.

## متى لا نستخدمه؟

لا تستخدم `exclude` بدل تقسيم الوحدات. إن كانت الإعارات والأعضاء عالمين مختلفين، اربط كل Middleware بمتحكمه. الاستثناء أداة طرح صغيرة.

## مقارنة

| الوسيط | يقبل | الأثر |
| --- | --- | --- |
| `forRoutes('loans')` | نص | ذلك المسار |
| `forRoutes({ path, method })` | `RouteInfo` | مسار وفعل |
| `forRoutes(LoansController)` | صنف | كل مساراته |
| `exclude(...)` | نص أو `RouteInfo` | طرح من المجموعة |

## تمارين

### تمرين سهل

أين يقع `exclude` في السلسلة بالنسبة إلى `forRoutes` في مثال التوثيق؟

<details>
<summary>الحل</summary>

قبله: `apply(...).exclude(...).forRoutes(...)`.

</details>

### تمرين متوسط

تريد `traceLoan` على كل `LoansController` ما عدا `DELETE /loans/:id`. ماذا تضع في `exclude`؟

<details>
<summary>الحل</summary>

`{ path: 'loans/:id', method: RequestMethod.DELETE }` مع ضبط `path` ليطابق البادئة الفعلية للمتحكم، ثم `forRoutes(LoansController)`.

</details>

### تمرين متقدم

النمط `reports/*rest` لم يعمل على `GET /reports`. ما تعديل المسار الذي تذكره الصفحة؟

<details>
<summary>الحل</summary>

`reports/{*rest}` حتى يصبح الذيل اختياريًا فيطابق المسار الأب والابن معًا. من غير الأقواس يُشترط وجود تتمة بعد الشرطة.

</details>

## أسئلة مراجعة

- ما الأشكال التي يقبلها `forRoutes`؟
- ماذا يفعل `exclude` إن كان المسار أصلًا خارج المجموعة؟
- كيف تكتب نمطًا يطابق بادئة وأي ذيل، بما فيه الذيل الفارغ؟
- لماذا ربط صنف المتحكم أمتن من سرد النصوص؟

## الخلاصة

`forRoutes` تبني المجموعة و `exclude` تطرح منها. الصنف أفضل من النص حين تتغير المسارات مع المتحكم. النمط المسمّى `*splat` وشكله الاختياري `{*splat}` هما نحو الصفحة الحالية.

## ماذا بعد؟

الدرس التالي يرتّب هذه السلاسل مع `app.use` ومع وحدات عدة: من يعمل أولًا حين يجتمع الجميع.

## المصادر الرسمية

- [Middleware](https://docs.nestjs.com/middleware)
- [Request lifecycle](https://docs.nestjs.com/faq/request-lifecycle)
