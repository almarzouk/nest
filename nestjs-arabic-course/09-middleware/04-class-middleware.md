# Class Middleware و NestMiddleware

## الهدف من هذا الدرس

كتابة Middleware كصنف يطبّق `NestMiddleware` ويحقن مزوّدًا من الوحدة نفسها.

## المشكلة التي يحلها

معرّف الطلب السهل دالة. ختمًا يعتمد على ساعة قابلة للاستبدال في الاختبار، أو على خدمة تقرأ إعدادًا، لا تعيش في دالة مغلقة على `Date.now`.

## الفكرة ببساطة

صنف عليه `@Injectable()` وفيه `use(req, res, next)`. الواجهة `NestMiddleware` تثبّت هذا التوقيع. الاعتماديات تأتي من `constructor` كما في أي مزوّد.

## التعريف التقني

```ts
interface NestMiddleware<TRequest = any, TResponse = any> {
  use(req: TRequest, res: TResponse, next: (error?: any) => void): void | Promise<void>;
}
```

التوثيق يسمح بأن تكون `use` غير متزامنة. الوعد المرفوض يصل إلى طبقة الاستثناءات إن كان الربط عبر `MiddlewareConsumer`.

الحقن يعمل لمزوّدات الوحدة نفسها، بالطريقة التي تُحقن بها المتحكمات. الصنف لا يُسجَّل في `providers` كي «يصبح Middleware». التسجيل الذي يشغّله على الطلب هو `consumer.apply(ShelfClockMiddleware)` في درس `configure`. الحاوية تستطيع بناءه لأن Nest يدير أصناف Middleware المطبَّقة هناك.

`app.use(new ShelfClockMiddleware(...))` يمر خارج الحاوية. لذلك الساعة التي تريد حقنها تُربَط من الوحدة، لا من `main.ts`.

## Mental Model

```text
ShelfClock          مزوّد في providers
ShelfClockMiddleware  constructor(clock)
apply(ShelfClockMiddleware)   Nest يبني الصنف ويحقن الساعة
use()               نفس عقد next
```

الصنف طبقة رقيقة حول خدمة. القاعدة «كيف نقيس الوقت» تعيش في `ShelfClock`. `use` تلصق الناتج على الرد.

## أبسط مثال

```ts
import { Injectable, type NestMiddleware } from '@nestjs/common';
import type { NextFunction, Request, Response } from 'express';
import { ShelfClock } from './shelf-clock.js';

@Injectable()
export class ShelfClockMiddleware implements NestMiddleware {
  constructor(private readonly clock: ShelfClock) {}

  use(req: Request, res: Response, next: NextFunction): void {
    res.setHeader('x-shelf-time', this.clock.iso());
    next();
  }
}
```

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class ShelfClock {
  iso(): string {
    return new Date().toISOString();
  }
}
```

`ShelfClock` في `providers` الخاصة بالوحدة التي تملك الصنف. الاختبار يستبدل `ShelfClock` بكلاس يرجع `2026-10-05T06:00:00.000Z`.

## شرح المثال

- `implements NestMiddleware` يجعل نسيان `use` خطأ TypeScript.
- `@Injectable()` مطلوب حتى تملأ الحاوية `constructor`.
- `private readonly clock` يحفظ الاعتماد كما في الخدمات.
- `res.setHeader` أثر جانبي على الرد. `next()` يسلم السلسلة.
- لا تضع `@Controller` هنا. هذا الصنف ليس مسارًا.

## ماذا يحدث داخليًا؟

عند الإقلاع يرى Nest الصنف داخل `apply`. يبنيه من حاوية الوحدة ويحل `ShelfClock`. عند الطلب يستدعي `use` لا الدالة التي صدّرتها أنت باسم آخر.

إن كانت `use` ترجع `Promise`، الرفض يُعامَل كاستثناء في طبقة الاستثناءات للربط عبر المستهلك. الفلاتر التي تراه هي العامة، لأن المسار لم يُختر بعد. هذا قيد الدورة، مستقل عن كون الكتابة صنفًا.

`@UseFilters()` على الصنف غير مدعوم. لا تبحث عن هذا السطر في الأمثلة الرسمية.

المزوّد يجب أن يكون مرئيًا في الوحدة التي تطبّق Middleware. خدمة مُصدَّرة من وحدة مستوردة تُحقن إن كانت الوحدة المستوردة ضمن `imports`، بنفس قواعد الحقن التي تعرفها. خدمة غير مصدَّرة من وحدة أخرى تبقى مخفية.

## مثال عملي

الاختبار يريد وقتًا ثابتًا على `GET /copies`. تستبدل `ShelfClock` في وحدة الاختبار. `ShelfClockMiddleware` لا يتغير. طلب حقيقي في التطوير يقرأ الساعة الحقيقية. المعالج يعيد نسخة الكتاب، والترويسة `x-shelf-time` تأتي من الصنف. هكذا يبقى الوقت قابلًا للاستبدال من غير معامل إضافي على كل مسار.

## أخطاء شائعة

- نسيان `@Injectable()` ثم الدهشة أن `clock` قيمته `undefined`.
- وضع منطق الساعة داخل `use` ثم عجز الاختبار عن تثبيت الوقت.
- تمرير `new ShelfClockMiddleware(new ShelfClock())` إلى `app.use` وتسمية ذلك حقنًا. البناء اليدوي يقطع الحاوية.
- جعل `use` تنهي كل الطلبات بـ `res.json` حتى «تتأكد أنها عملت». المعالج لن يعمل.

## متى نستخدمه؟

عند وجود اعتماد واحد على الأقل تريد استبداله في الاختبار أو مشاركته مع خدمة: ساعة، إعداد، مسجّل.

## متى لا نستخدمه؟

دالة من سطرين بلا `constructor`. التوثيق يطلب functional middleware هناك حتى لا يصنع الملف صنفًا فارغًا.

## مقارنة

| | `NestMiddleware` | مزوّد عادي |
| --- | --- | --- |
| الدالة التي يستدعيها الإطار على الطلب | `use` | لا يستدعيها على كل طلب |
| من يبنيه | Nest عند `apply(Class)` | الحاوية عند الحقن |
| عقد المنصة | `req, res, next` | لا يعرفه |

## تمارين

### تمرين سهل

ما اسم الدالة التي يستدعيها Nest على الصنف عند وصول الطلب؟

<details>
<summary>الحل</summary>

`use`. ليست `handle` ولا `intercept`. الواجهة `NestMiddleware` تثبّت هذا الاسم.

</details>

### تمرين متوسط

`ShelfClock` غير مذكور في `providers` ولا في `exports` لوحدة مستوردة. ماذا يحدث عند الإقلاع؟

<details>
<summary>الحل</summary>

تفشل الحاوية في بناء `ShelfClockMiddleware` لأن الاعتماد غير متاح في تلك الوحدة. العلاج تسجيل الساعة في الوحدة المالكة، أو استيراد وحدة تصدّرها.

</details>

### تمرين متقدم

تريد أن تقرأ `use` علم ميزة من `ConfigService` قبل `next()`. أين يعيش الصنف، ولماذا يفشل `app.use` هنا؟

<details>
<summary>الحل</summary>

الصنف في الوحدة التي تستورد وحدة الإعداد وترى `ConfigService`. `apply(ShelfClockMiddleware)` من `configure`. `app.use` لا يسأل الحاوية، فـ `ConfigService` لن تُحقن.

</details>

## أسئلة مراجعة

- ماذا تطبّق الواجهة `NestMiddleware`؟
- من أين تأتي اعتماديات `constructor`؟
- لماذا لا يكفي `app.use` لصنف يحتاج خدمة؟
- ماذا يعني أن ترجع `use` وعدًا مرفوضًا؟

## الخلاصة

الصنف `@Injectable()` مع `use` هو Middleware حين تحتاج الحقن. الواجهة تثبت التوقيع. الحاوية تبنيه لأن الوحدة طبّقته عبر `apply`، لا لأنك وضعته في `providers` كمسار.

## ماذا بعد؟

الدرس التالي هو مكان الاستدعاء: `configure` و `MiddlewareConsumer` على وحدة تطبّق `NestModule`.

## المصادر الرسمية

- [Middleware](https://docs.nestjs.com/middleware)
