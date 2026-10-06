# configure و MiddlewareConsumer

## الهدف من هذا الدرس

ربط Middleware من داخل الوحدة عبر `configure(consumer)`، وفهم أن `MiddlewareConsumer` أداة سلسلة لا مسار HTTP.

## المشكلة التي يحلها

الصنف أو الدالة بلا تسجيل لا تعمل على أي طلب. وضعها في `providers` يسجّل مزوّدًا ولا يضعها في سلسلة الطلب. المكان الذي يقوله التوثيق هو `configure`.

## الفكرة ببساطة

الوحدة تطبّق `NestModule` وتعرّف `configure(consumer)`. داخلها `consumer.apply(...).forRoutes(...)`. السلسلة تُقرأ عند الإقلاع.

## التعريف التقني

`NestModule` واجهة فيها `configure(consumer: MiddlewareConsumer): void`، ويمكن أن تكون `async` إن احتجت `await` قبل إتمام الربط.

`MiddlewareConsumer` يوفّر دوالًا تُسلسَل:

- `apply(...middleware)` تقبل صنفًا أو دالة أو عدة وسطاء في استدعاء واحد.
- `forRoutes(...)` تحدد أين تُطبَّق. تفاصيل الأنواع في الدرس التالي.
- `exclude(...)` تستثني. الدرس التالي أيضًا.

Middleware لا يُذكر داخل مفاتيح `@Module()`. غيابه من `controllers` و `providers` مقصود.

عدة استدعاءات `apply` في الدالة نفسها تُسجَّل بالترتيب الذي كتبته. هذا ترتيب داخل الوحدة، قبل أن يُدمج مع ترتيب الوحدات في دورة الطلب.

## Mental Model

```text
@Module({ controllers, providers })   من يملك المسارات والخدمات
implements NestModule                 يعد بأن فيه configure
configure(consumer)                   يبني سلسلة هذه الوحدة
apply ثم forRoutes                    يربط سلسلة بمسارات
```

`consumer` ليس `req`. هو مسجّل يعمل مرة عند الإقلاع.

## أبسط مثال

```ts
import {
  Module,
  type MiddlewareConsumer,
  type NestModule,
} from '@nestjs/common';
import { LoansController } from './loans.controller.js';
import { ShelfClock } from './shelf-clock.js';
import { ShelfClockMiddleware } from './shelf-clock.middleware.js';
import { traceLoan } from './trace-loan.middleware.js';

@Module({
  controllers: [LoansController],
  providers: [ShelfClock],
})
export class LoansModule implements NestModule {
  configure(consumer: MiddlewareConsumer): void {
    consumer.apply(traceLoan).forRoutes(LoansController);
    consumer.apply(ShelfClockMiddleware).forRoutes(LoansController);
  }
}
```

على طلب يطابق المتحكم تعمل `traceLoan` ثم `ShelfClockMiddleware`.

## شرح المثال

- `implements NestModule` يجبر TypeScript على وجود `configure`.
- الاستدعاء الأول يربط الدالة. الثاني يربط الصنف. Nest يبني الصنف ويحقن `ShelfClock` لأن الساعة في `providers`.
- فصل الاستدعاءين أوضح من دمجهما حين تريد قراءة الترتيب. الدمج `apply(traceLoan, ShelfClockMiddleware)` يعطي الترتيب نفسه.
- لا سطر `providers: [ShelfClockMiddleware]`. التطبيق عبر `apply` هو التسجيل الذي يخص الطلب.

## ماذا يحدث داخليًا؟

1. الإقلاع يمشي على الوحدات.
2. كل وحدة تطبّق `NestModule` تُستدعى `configure` الخاصة بها.
3. `MiddlewareModule` يحتفظ بالسلاسل والمسارات.
4. عند الطلب تُختار السلاسل المطابقة بعد Middleware العام القادم من `app.use`.
5. إن كانت `configure` غير متزامنة، ينتظر الإطار الوعد قبل اعتبار الربط اكتمل.

الوحدة التي لا تعرّف `configure` لا تساهم في هذه الطبقة. مساراتها تبقى بلا Middleware الوحدة حتى لو كان ملف الدالة موجودًا في المجلد.

استيراد `LoansController` في ملف الوحدة طبيعي: `forRoutes` يحتاج الصنف. لاحقة `.js` تبقى على الاستيراد المحلي.

## مثال عملي

`CopiesModule` تملك `CopiesController` وتريد التتبع على النسخ فقط. `configure` فيها تستدعي `apply(traceLoan).forRoutes(CopiesController)`. وحدة الأعضاء لا تعرّف `configure`، فطلبات `GET /members` لا تمر على `traceLoan`. الحدود ظاهرة في ملف كل وحدة، لا في `main.ts` طويل.

## أخطاء شائعة

- كتابة `configure` من غير `implements NestModule`. طبّق الواجهة حتى يراك TypeScript والإطار معًا.
- وضع `consumer.apply` داخل `onModuleInit`. الربط المتأخر عن `configure` ليس المسار الموثّق.
- استدعاء `forRoutes` من غير `apply`. السلسلة تبدأ بما يُطبَّق.
- جعل `configure` في `AppModule` تضم كل وسطاء التطبيق. الوسط الذي يخص الإعارات يعيش في `LoansModule` حتى تبقى الوحدة مالكة لمسارها.

## متى نستخدمه؟

في كل وحدة تريد Middleware على مساراتها: سطر `configure` بجانب `@Module`.

## متى لا نستخدمه؟

لدالة تعمل على كل طلب بلا حقن. `app.use` في `main.ts` أقصر، والتوثيق يبقيه لذلك. ما زلت تستطيع `forRoutes('*')` من الجذر إن أردت صنفًا عامًا يدخل الحاوية، وهذا تلميح الصفحة للبديل عن `app.use`.

## مقارنة

| | `configure` | `app.use` |
| --- | --- | --- |
| أين | صنف الوحدة | `main.ts` بعد `create` |
| ما يُمرَّر | `MiddlewareConsumer` | دالة إلى المنصة |
| الحقن | متاح لأصناف `apply(Class)` | غير متاح |
| التقييد | `forRoutes` و `exclude` | مسار المنصة أو الكل |

## تمارين

### تمرين سهل

ما اسم الدالة التي تستلم `MiddlewareConsumer`؟

<details>
<summary>الحل</summary>

`configure`. تعيش على صنف الوحدة الذي يطبّق `NestModule`.

</details>

### تمرين متوسط

لماذا وُضعت `ShelfClock` في `providers` بينما `ShelfClockMiddleware` وُضع في `apply`؟

<details>
<summary>الحل</summary>

الساعة مزوّد يُحقن. الصنف Middleware يُسجَّل على سلسلة الطلب عبر `apply`. وضعه في `providers` وحده لا يجعل `use` تعمل على الطلب.

</details>

### تمرين متقدم

`configure` تحتاج أن تقرأ قائمة مسارات من ملف قبل الربط. ما التعديل على توقيع الدالة؟

<details>
<summary>الحل</summary>

`async configure(consumer: MiddlewareConsumer): Promise<void>` ثم `await` قبل `apply`. التوثيق يسمح بأن تكون `configure` غير متزامنة.

</details>

## أسئلة مراجعة

- أي واجهة تطبّقها الوحدة التي تربط Middleware؟
- ماذا يفعل `apply` مقارنة بـ `forRoutes`؟
- هل يُذكر Middleware داخل كائن `@Module`؟
- ما ترتيب استدعاءين متتاليين لـ `apply` في الملف نفسه؟

## الخلاصة

`configure` هي لحظة ربط Middleware بالوحدة. `MiddlewareConsumer` يجمع `apply` مع تحديد المسار. الكود الذي لا يمر من هنا لا يقف في سلسلة طلبات تلك الوحدة.

## ماذا بعد؟

الدرس التالي يفصّل وسيطَي التحديد: `forRoutes` و `exclude`، من النص إلى صنف المتحكم.

## المصادر الرسمية

- [Middleware](https://docs.nestjs.com/middleware)
