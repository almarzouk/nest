# Global و Controller و Route

## الهدف من هذا الدرس

تعريف مستويات الربط الثلاثة، واتجاه كل عائلة عند التنفيذ.

## المشكلة التي يحلها

`@UseGuards` على الدالة و `APP_GUARD` و `useGlobalGuards` تبدو ثلاثة أساليب للأداة نفسها. هي مستويات في القائمة التي ثبتناها: العام خارج الصنف، و Controller على الصنف، والمسار على الدالة. Middleware لا تستخدم هذه الزينة الثلاث؛ ربطها `app.use` أو وحدة. خلط النموذجين يضع ختمًا عامًا في مكان مسار.

## الفكرة ببساطة

Guard و Interceptor و Pipe و Exception Filter تُربَط عامًا أو على Controller أو على مسار. العام دالة تطبيق مثل `app.useGlobalGuards()` أو مزود `APP_GUARD` ومثيله `APP_PIPE` و `APP_INTERCEPTOR` و `APP_FILTER`.

اتجاه التنفيذ ليس واحدًا لكل العائلات. Guards ودخول Interceptors و Pipes من العام إلى المسار. خروج Interceptors من المسار إلى العام. المرشحات من المسار إلى العام. داخل المستوى ترتيب الربط، ويُعكس في خروج Interceptor لأنه `Observable`.

## التعريف التقني

| العائلة | الرمز العام | على الصنف | على الدالة |
| --- | --- | --- | --- |
| Guard | `APP_GUARD` أو `useGlobalGuards` | `@UseGuards` | `@UseGuards` |
| Interceptor | `APP_INTERCEPTOR` أو `useGlobalInterceptors` | `@UseInterceptors` | `@UseInterceptors` |
| Pipe | `APP_PIPE` أو `useGlobalPipes` | `@UsePipes` | `@UsePipes` أو على المعامل |
| Exception Filter | `APP_FILTER` أو `useGlobalFilters` | `@UseFilters` | `@UseFilters` |

المعامِل مستوى أخص من زينة الدالة لـ Pipes فقط، ويعمل بعد زينة المسار كما في درس Pipes.

Middleware: `app.use` عام من غير `APP_*`. ربط الوحدة عبر مستهلك Middleware في فصل Middleware، ويُرتَّب `@Global()` ثم الجذر ثم البعد عن الجذر. لا `@UseMiddleware` على الدالة في هذا النموذج.

عدة مزودات بالرمز نفسه، مثل `APP_GUARD` مرتين، مستوى عام واحد بترتيب التسجيل.

## Mental Model

```text
عام: كل المسارات التي تدخل التطبيق
Controller: دوال هذا الصنف
Route: هذه الدالة
معامل: هذا الوسيط، لـ Pipes
```

السهم عند التشغيل يعتمد على العائلة لا على عرض المستوى في الملف.

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { LibraryOpenGuard } from './library-open.guard.js';

@Module({
  providers: [{ provide: APP_GUARD, useClass: LibraryOpenGuard }],
})
export class AppModule {}
```

```ts
@UseGuards(MemberGuard)
@Controller('loans')
export class LoansController {
  @UseGuards(PatchGuard)
  @Patch(':id')
  update() {
    return { ok: true };
  }
}
```

## شرح المثال

- `LibraryOpenGuard` عام لأنه `APP_GUARD`، فيسبق `MemberGuard` ثم `PatchGuard`.
- `GET` على Controller آخر ما زال يمر على `LibraryOpenGuard` ولا يمر على `MemberGuard`.
- `useClass` هنا وصفة Provider من الفصل السابق. الحارس يُحقن كأي صنف مسجّل.
- `@Patch` بلا حارس دالة كان سيكتفي بالعام وحارس الصنف.

## ماذا يحدث داخليًا؟

عند الإقلاع تسجل Nest الزينة على الصنف والدالة، وتجمع مزودات `APP_*` من الوحدات المستوردة في الرسم. الطلب بعد Middleware يبني قائمة تلك العائلة حسب الاتجاه الخاص بها.

خروج Interceptor لا يعيد قراءة الزينة من الأعلى. الأغلفة التي دُخلت من العام إلى المسار تُحسم من المسار إلى العام.

مرشح `APP_FILTER` يبقى آخر الفحص رغم أنه «عام» في ملف الجذر. كلمة عام تصف عرض الربط لا أنه أول من يعمل.

`useGlobalPipes` في `main` بعد `create` مستوى عام أيضًا، مثل المزود. لا تجمع الاثنين على المنطق نفسه من غير قصد فتعمل Pipe مرتين على الجولة العامة.

## مثال عملي

خريطة إعارة واحدة:

```text
APP_GUARD LibraryOpenGuard
APP_PIPE GeneralValidationPipe
APP_FILTER FallbackFilter
@UseInterceptors(TimingInterceptor) على LoansController
@UsePipes(RouteSpecificPipe) على PATCH
@UseFilters(LoanPatchFilter) على PATCH
@Param('id', ParseIntPipe)
```

الدخول: `LibraryOpenGuard`، ثم دخول `TimingInterceptor`، ثم `GeneralValidationPipe` على المعاملات من الأخير إلى الأول، ثم `RouteSpecificPipe` بالجولة نفسها، ثم `ParseIntPipe`. بعد الخدمة خروج `TimingInterceptor`. عند الرمي: `LoanPatchFilter` قبل `FallbackFilter`.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `APP_FILTER` مع توقع أنه يسبق مرشح الدالة | الفحص من المسار صعودًا |
| Pipe عامة عبر المزود وعبر `useGlobalPipes` للقاعدة نفسها | جولتان عامتان |
| البحث عن `@UseGuards` لتأثير Middleware | Middleware خارج المستويات الثلاثة |
| ربط حارس مسار لأثر يراد به كل التطبيق | المسارات الأخرى بلا الحارس |

## متى نستخدمه؟

العام لقاعدة كل الطلبات بعد Middleware. Controller لمجموعة المورد. المسار لاستثناء دالة واحدة. المعامل لتحويل وسيط. اختر أضيق مستوى يكفي حتى لا تُفرَض القاعدة على مسار لا يحمل المعامل.

## متى لا نستخدمه؟

لا ترفع Pipe معامل إلى `APP_PIPE` حتى «تضمن الترتيب». الترتيب بين المستويات ثابت، والعام الذي يتوقع جسمًا على `GET` بلا جسم يفشل طلبات لا تخصه. ولا تربط منطقًا قبل اختيار المسار بـ `APP_GUARD`؛ ذلك `app.use`.

## مقارنة

| العائلة | من أين تبدأ التنفيذ |
| --- | --- |
| Guard | العام ثم Controller ثم المسار |
| Interceptor دخولًا | العام ثم Controller ثم المسار |
| Interceptor خروجًا | المسار ثم Controller ثم العام |
| Pipe | العام ثم Controller ثم المسار ثم المعامل |
| Exception Filter | المسار ثم Controller ثم العام |
| Middleware | `app.use` ثم الوحدات، لا زينة الدالة |

## تمارين

### تمرين سهل

ماذا يجعل `provide: APP_GUARD` الحارس عامًا؟

<details><summary>الحل</summary>

الرمز المحجوز يسجل الحارس على التطبيق، فيسبق `@UseGuards` على الصنف وعلى الدالة.

</details>

### تمرين متوسط

لماذا `FallbackFilter` في المثال يعمل بعد `LoanPatchFilter` رغم أن الأول في الجذر؟

<details><summary>الحل</summary>

عرض الربط عام، واتجاه المرشحات من المسار إلى العام. مرشح `PATCH` يُسأل أولًا، والعام آخرًا إن لم يُمسك الاستثناء.

</details>

### تمرين متقدم

كيف تفرض ختمًا قبل اختيار `LoansController` من غير `APP_GUARD`؟

<details><summary>الحل</summary>

Middleware عبر `app.use` أو ربط وحدة يعمل قبل Guards. `APP_GUARD` بعد تلك المرحلة وبعد أن صار Controller معروفًا.

</details>

## أسئلة مراجعة

- ما رموز المزود العامة الأربعة المذكورة؟
- أي عائلة تبدأ من المسار رغم أن ربطها قد يكون في الجذر؟
- أين يقع Pipe المعامل بين المستويات؟

## الخلاصة

المستويات الثلاثة عرض الربط: التطبيق، الصنف، الدالة، ومعامل Pipe أخص منها. اتجاه التشغيل ثابت لكل عائلة: المنع والدخول والتحويل من العام نزولًا، والخروج والمرشحات من الدالة صعودًا. Middleware تبقى `app.use` والوحدات.

## ماذا بعد؟

متى تتوقف القائمة قبل الاستجابة الطبيعية: متى يُتخطى باقي المسار.

## المصادر الرسمية

- NestJS — Request lifecycle: https://docs.nestjs.com/faq/request-lifecycle
