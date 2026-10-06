# Global Filters

## الهدف من هذا الدرس

تسجيل فلتر على التطبيق كله، وفهم أنه الوحيد الذي يرى استثناء Middleware.

## المشكلة التي يحلها

نسخ `@UseFilters` على كل متحكم تنسى مسارًا. واستثناء من `traceLoan` لا يمر على فلتر `TicketsController` أصلًا. الشكل المشترك الأخير يحتاج ربطًا عامًا.

## الفكرة ببساطة

`app.useGlobalFilters(new DeskExceptionFilter())` أو مزوّد `APP_FILTER`. الربط العام يرى ما تبقى بعد فلاتر المسار والمتحكم، ويرى أيضًا استثناء Middleware لأن تلك الطبقة لا تملك فلتر مسار.

## التعريف التقني

بلا حقن:

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalFilters(new DeskExceptionFilter());
await app.listen(3000);
```

مع حقن، من `@nestjs/core`:

```ts
providers: [{ provide: APP_FILTER, useClass: DeskExceptionFilter }]
```

`new` في `main.ts` خارج الحاوية. `ConfigService` في الباني تحتاج `APP_FILTER`. يمكن تسجيل أكثر من `APP_FILTER`. الترتيب مع فلاتر المسار يُشرح في الدرس التالي: الأخص أولًا، والعام أخيرًا بين المستويات. عدة فلاتر عامة تتبع ترتيب التسجيل عند البحث داخل المستوى العام.

في تطبيق هجين، `useGlobalFilters` قد لا يشمل الناقل الآخر إلا بضبط لاحق. في HTTP يشمل المتحكمات.

استثناء Middleware، كما تقول صفحة Middleware ودورة الطلب: الفلاتر العامة فقط، سواء المدمج أو ما سجّلته بـ `useGlobalFilters` أو `APP_FILTER`. `@UseFilters` على صنف Middleware غير مدعوم. فلتر المتحكم لا يُختار قبل تحديد المسار.

`app.use` على المنصة قد يمر خطأه خارج `MiddlewareModule`. الصفحة تفضّل الرمي من Middleware مربوط بـ `MiddlewareConsumer` حتى تصل طبقة Nest.

الفلتر العام ما زال ملزمًا بقاعدة `@Catch`: لا تحوّل `HttpException` إلى `500`.

## Mental Model

```text
رمي من reserve           قد يمسكه فلتر المسار أولًا، وإلا العام
رمي من traceLoan         العام فقط
useGlobalFilters(new)    بلا حقن
APP_FILTER               مع حقن
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';
import { DeskExceptionFilter } from './desk-exception.filter.js';

@Module({
  providers: [{ provide: APP_FILTER, useClass: DeskExceptionFilter }],
})
export class TicketsModule {}
```

`DeskExceptionFilter` هنا هو صنف الدرس السابق الذي يفرّع `HttpException`. تسجيله في وحدة التذاكر يجعله عامًا على التطبيق كله رغم مكان الملف، كما في تلميح `APP_GUARD`. الأفضل أن يعيش في الوحدة التي تملكه.

## شرح المثال

- `useClass` يبني من الحاوية.
- الزينة على المتحكم غير لازمة لهذا الربط.
- الصحة و `POST /tickets` و Middleware الوحدات تدخل النطاق.
- إن وُجد `@UseFilters(SeatLogFilter)` على دالة الاسترداد، ذلك الفلتر يُجرَّب أولًا لتلك الدالة. العام يبقى لما لا يُمسك.

## ماذا يحدث داخليًا؟

الإقلاع يجمع `APP_FILTER`. عند استثناء غير ممسوك يبحث الإطار من الأخص إلى الأعم. إن لم يمسك مسار ولا متحكم، يصل العام. عند استثناء Middleware يبدأ البحث من العام مباشرة.

الفلتر المدمج ما زال موجودًا كسلوك أساس إن لم يمسك فلترك الاستثناء. `@Catch()` الفارغ يمسك قبل أن تحتاج المدمج، لذلك يجب أن يكتب ردًا صحيحًا بنفسه.

حارس المسار الذي يرمي `UnauthorizedException` لا يذهب إلى العام إن مسكه فلتر على الدالة. لا تكرر كتابة الجسم في العام وفي المسار إلا بوراثة.

## مثال عملي

`traceLoan` يرمي `BadRequestException` لمعرف تتبع أطول من الحد. فلتر على `TicketsController` لا يغيّر الرد. بعد `APP_FILTER` يظهر جسم `DeskExceptionFilter` مع الرمز `400`. الاختبار ينادي مسارًا يمر على Middleware التتبع بترويسة طويلة ويتوقع `400` لا `500`. ثم يطلب مقعدًا تالفًا على `POST` ويتوقع `400` أيضًا من الأنبوب عبر الفلتر نفسه، ما دام لا يوجد فلتر مسار يطمس الرمز.

## أخطاء شائعة

- توقع `@UseFilters` على المتحكم أن يلتقط Middleware.
- `useGlobalFilters(new Filter())` مع مزوّد في الباني.
- فلتر عام يكتب `500` لكل شيء فيحوّل أخطاء التتبع السليمة.
- تسجيل الفلتر مرتين، في `main.ts` وفي `APP_FILTER`، فتتكرر الآثار الجانبية مثل السجل.

## متى نستخدمه؟

لشكل خطأ أخير مشترك، ولأي استثناء قد يخرج من Middleware، مع الإبقاء على رموز `HttpException`.

## متى لا نستخدمه؟

لشكل يخص دالة واحدة فقط. ذلك `@UseFilters` على الدالة حتى لا تغيّر الصحة. ولا كبديل عن رمي النوع الصحيح من الخدمة.

## مقارنة

| الربط | يرى `reserve` | يرى Middleware | الحقن |
| --- | --- | --- | --- |
| على الدالة | إن لم يُمسك أقرب منه | لا | صنف الزينة نعم |
| على المتحكم | نعم لدواله | لا | نعم |
| `useGlobalFilters(new)` | لما يصل إلى العام | نعم | لا |
| `APP_FILTER` | لما يصل إلى العام | نعم | نعم |

## تمارين

### تمرين سهل

أي ربط يرى استثناء `traceLoan`؟

<details>
<summary>الحل</summary>

فلتر عام: `useGlobalFilters` أو `APP_FILTER`، بالإضافة إلى الفلتر المدمج. فلتر المتحكم لا يراه.

</details>

### تمرين متوسط

لماذا تفشل `ConfigService` داخل فلتر بُني بـ `new` في `main.ts`؟

<details>
<summary>الحل</summary>

البناء خارج الوحدات. سجّل `APP_FILTER` مع `useClass` في وحدة ترى الإعداد.

</details>

### تمرين متقدم

فلتر المسار يمسك `SeatTakenException` فقط، والعام يمسك الباقي بفرع HTTP. من يكتب رد المقعد المبيع، ومن يكتب رد معرف التتبع التالف في Middleware؟

<details>
<summary>الحل</summary>

فلتر المسار يكتب `409` للمقعد لأنه أخص ويطابق النوع. معرف التتبع في Middleware لا يرى فلتر المسار، فيكتبه العام، ويجب أن يبقى `400` لا `500`.

</details>

## أسئلة مراجعة

- ما رمز المزود للفلتر المحقون؟
- لماذا لا يكفي `@UseFilters` على المتحكم لخطأ Middleware؟
- ماذا يجب أن يفعل الفلتر العام بـ `HttpException`؟
- متى تفضّل الربط على الدالة؟

## الخلاصة

الفلتر العام شبكة أخيرة وشبكة Middleware. الحقن عبر `APP_FILTER`. الأخص على المسار يسبق العام حين يكون المسار مختارًا. الرمز الذي رماه `HttpException` يبقى كما هو.

## ماذا بعد؟

الدرس التالي يرتّب البحث: مسار، ثم متحكم، ثم عام، ولماذا لا يُستدعى الثاني بعد الأول.

## المصادر الرسمية

- [Exception filters](https://docs.nestjs.com/exception-filters)
- [Request lifecycle](https://docs.nestjs.com/faq/request-lifecycle)
- [Middleware](https://docs.nestjs.com/middleware)
