# Functional Middleware

## الهدف من هذا الدرس

كتابة Middleware كدالة بلا صنف، وتمريرها إلى `apply` عندما لا توجد اعتماديات.

## المشكلة التي يحلها

صنف فيه `use` واحدة وسطر واحد يخفي أن العمل دالة. التوثيق يسمي هذا الشكل functional middleware ويطلبه حين لا تحتاج `constructor`.

## الفكرة ببساطة

تصدّر دالة باسم واضح بمعاملات `(req, res, next)`. الوحدة تستدعي `consumer.apply(اسمالدالة)`.

## التعريف التقني

الدالة لا تطبّق واجهة. لا `@Injectable()`. Nest يقبلها لأن `apply` يقبل دالة Middleware كما يقبل صنفًا.

الشرط الوحيد هو عقد Express: إن لم تنهِ الرد فنادِ `next()`. الدالة تستطيع أن تكون `async` إن كان العمل غير متزامن، والأوضح في الأمثلة القصيرة الإبقاء على الشكل المتزامن ونقل العمل الذي يحتاج خدمة إلى صنف.

عدة دوال في استدعاء واحد تعمل بالترتيب المكتوب:

```ts
consumer.apply(traceLoan, stampArrival).forRoutes(LoansController);
```

`traceLoan` ثم `stampArrival`. هذا ترتيب `apply`، ودرس الترتيب يضع هذا السطر داخل ترتيب أوسع.

## Mental Model

```text
trace-loan.middleware.ts     دالة مُصدَّرة
loans.module.ts              apply(traceLoan)
لا providers                 الدالة ليست مزوّدًا يُحقن
```

ملف الدالة صغير. الاختبار يستدعيها بكائنات طلب ورد مزيفة وينتظر أن `next` استُدعيت مرة.

## أبسط مثال

```ts
import { randomUUID } from 'node:crypto';
import type { NextFunction, Request, Response } from 'express';

export function traceLoan(
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  const incoming = req.header('x-request-id');
  const requestId = incoming && incoming.length > 0 ? incoming : randomUUID();
  res.setHeader('x-request-id', requestId);
  next();
}
```

```ts
import { Module, type MiddlewareConsumer, type NestModule } from '@nestjs/common';
import { LoansController } from './loans.controller.js';
import { traceLoan } from './trace-loan.middleware.js';

@Module({
  controllers: [LoansController],
})
export class LoansModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer.apply(traceLoan).forRoutes(LoansController);
  }
}
```

## شرح المثال

- `req.header` يقرأ الترويسة إن أرسلها البوابة. القيمة الفارغة تُستبدل بـ `randomUUID()`.
- الرد يعيد المعرّف حتى يضعه العميل في تذكرة الدعم.
- `apply` يأخذ الدالة نفسها لا اسم ملف نصي.
- لاحقة `.js` في الاستيراد لأن المشروع ESM كما يولّده NestJS 12.
- `forRoutes(LoansController)` يحدّد أين تعمل. تفاصيل `exclude` درس لاحق. هنا يكفي أن الدالة لا تعمل على التطبيق كله من تلقاء نفسها.

## ماذا يحدث داخليًا؟

Nest يضع الدالة في سلسلة المسارات المطابقة. عند طلب يطابق `LoansController` تُستدعى `traceLoan` بمعاملات المنصة. `next()` يمرر إلى Middleware التالي ثم يخرج الطلب من هذه الطبقة نحو Guards.

الدالة لا تُنشأ عبر الحاوية. لا تستطيع كتابة `constructor`. إن احتجت `ConfigService` فذلك صنف في الدرس التالي، أو قراءة محدودة جدًا لا تناسب الاختبار. الإغلاق على متغير خارجي يعيد أسلوب Express ويصعّب استبدال الاعتماد في الاختبار.

`configure` يمكن أن تكون `async`. دالة التتبع هذه لا تنتظر شيئًا، فـ `configure` تبقى متزامنة.

## مثال عملي

على مسار `POST /loans` يصل طلب من البوابة بترويسة `x-request-id: 9f2`. الدالة تبقي القيمة وتكتبها على الرد. طلب من أداة محلية بلا الترويسة يحصل على UUID جديد. `LoansController` لا يقرأ الترويسة. السجل في طرف آخر يقرأ الرد. هكذا يبقى التتبع خارج قاعدة الإعارة.

## أخطاء شائعة

- تسجيل الدالة في `providers`. هي ليست مزوّدًا. مكانها `apply`.
- استدعاء `traceLoan()` داخل `apply` من غير معاملات المنصة. `apply` يريد الدالة، والمنصة تستدعيها. استدعِ أنت فقط إن كانت الحزمة مصنعًا يرجع دالة، مثل `consumer.apply(factory())`.
- كتابة منطق الدور داخل الدالة لأنها «أسهل من Guard». الدالة لا ترى metadata المسار.
- نسيان تصدير الدالة ثم تمرير اسم غير معرّف إلى `apply`.

## متى نستخدمه؟

دالة قصيرة بلا اعتماديات: معرّف طلب، ختم، أو غلاف حول Middleware جاهز من حزمة Express.

## متى لا نستخدمه؟

حين تحتاج خدمة من الحاوية، أو حالة مُدارة عبر `constructor`. الصنف هو الشكل الذي يحقن.

## مقارنة

| | Functional | Class |
| --- | --- | --- |
| الشكل | دالة مُصدَّرة | `@Injectable()` و `use` |
| الحقن | غير متاح | `constructor` |
| التسجيل | `apply(fn)` | `apply(Class)` |
| توجيه التوثيق | عند غياب الاعتماديات | عند الحاجة إليها |

## تمارين

### تمرين سهل

لماذا لا نضع `@Injectable()` فوق `traceLoan`؟

<details>
<summary>الحل</summary>

لأنها دالة لا صنف. Nest يستدعيها مباشرة. الزينة والحقن للصنف الذي يطبّق `NestMiddleware`.

</details>

### تمرين متوسط

تريد تشغيل `traceLoan` ثم دالة `stampArrival` على المتحكم نفسه. ماذا تكتب داخل `apply`، وأي دالة ترى الطلب أولًا؟

<details>
<summary>الحل</summary>

`consumer.apply(traceLoan, stampArrival).forRoutes(LoansController)`. الأولى في القائمة، `traceLoan`، تعمل أولًا ثم `stampArrival`.

</details>

### تمرين متقدم

`traceLoan` تحتاج أن ترفض المعرّفات الأطول من 80 حرفًا برمز `400`. أبقِها دالة. ماذا ترمي، ومن في الفلاتر يرى الاستثناء؟

<details>
<summary>الحل</summary>

ترمي `BadRequestException` من `@nestjs/common` قبل `next()`، أو تمرر الخطأ إلى `next(error)`. لأن الربط عبر `MiddlewareConsumer`، طبقة الاستثناءات تستقبل الرمي. الذي يشكّل الرد من الفلاتر المخصصة هو الفلتر العام فقط.

</details>

## أسئلة مراجعة

- متى تفضّل الدالة على الصنف؟
- ماذا يُمرَّر إلى `apply`؟
- هل تُذكر الدالة في `providers`؟
- ما ترتيب دالتين داخل `apply` واحد؟

## الخلاصة

Functional middleware دالة بعقد `(req, res, next)` تُمرَّر إلى `apply` حين لا يوجد ما يُحقن. الاختبار يستدعيها مباشرة. الاعتماد على خدمة ينقلك إلى صنف.

## ماذا بعد؟

الدرس التالي يكتب الصنف: `NestMiddleware`، والدالة `use`، وحقن مزوّد من الوحدة نفسها.

## المصادر الرسمية

- [Middleware](https://docs.nestjs.com/middleware)
