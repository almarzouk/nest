# أمثلة عملية

ثلاثة طلبات على إعارة. كل مثال يتبع التسلسل الرسمي من غير إعادة ترتيب المراحل. الأصناف أسماء مواضع؛ بناؤها في الفصول التالية.

## مثال 1 — `GET /loans` ينجح

```text
app.use(requestId)
AuditModule @Global() -> stamp
AppModule -> accessLog
LoansModule -> loansMark

APP_GUARD LibraryOpenGuard
@UseGuards(MemberGuard) على LoansController

APP_INTERCEPTOR TimingInterceptor   دخول
@UseInterceptors(AuditInterceptor) على الصنف   دخول

لا Pipe على GET

LoansController.list -> LoansService.openList

خروج: AuditInterceptor ثم TimingInterceptor
استجابة 200 JSON
```

`requestId` قبل `stamp` لأن `app.use` قبل وحدات `@Global()`. `LibraryOpenGuard` قبل `MemberGuard`. `TimingInterceptor` يدخل قبل `AuditInterceptor` ويخرج بعده. لا سطر Exception Filter لأن `openList` أعادت قائمة.

## مثال 2 — `PATCH /loans/9` يرفضه الحارس

```text
Middleware كما في المثال 1، لأن المنع لم يبدأ بعد
LibraryOpenGuard ينجح
MemberGuard يعيد false

لا RouteSpecificPipe
لا ParseIntPipe على :id
لا LoansService.update
```

Nest ترمي استثناء الرفض. الفحص: مرشح على `PATCH` إن وُجد، ثم مرشح `LoansController`، ثم `APP_FILTER`. أول إمساك يكتب الاستجابة ويمنع المرشح التالي من رؤية الاستثناء نفسه. Pipe لم تعمل لأن موضعها بعد Guards.

## مثال 3 — جسم مقبول ثم رمي من الخدمة، ومسار CSV منفصل

`PATCH` يصل إلى الخدمة بعد الأنابيب:

```text
GeneralValidationPipe على آخر معامل ثم ما قبله
RouteSpecificPipe الجولة نفسها
ParseIntPipe على :id
update() -> LoansService.update ترمي
```

`catchError` في `TimingInterceptor` إن رجع قيمة، فالمرشح لا يعمل وتكمل الأغلفة الأوسع كنجاح. إن أعاد الرمي: `LoanPatchFilter` ثم مرشح الصنف ثم العام.

مسار آخر `GET /loans/export.csv` يستخدم `@Res()` من غير `passthrough` وينادي `res.send`. الحارس والـ Middleware ما زالا قبل الدالة. بعد `send` لا تسلسل JSON ولا خروج Interceptor على قيمة `return`.

```ts
import { Controller, Get, Res, UseGuards } from '@nestjs/common';
import type { Response } from 'express';
import { LoansService } from './loans.service.js';
import { MemberGuard } from './member.guard.js';

@UseGuards(MemberGuard)
@Controller('loans')
export class LoanExportController {
  constructor(private readonly loans: LoansService) {}

  @Get('export.csv')
  exportCsv(@Res() res: Response) {
    res.type('text/csv');
    res.send(this.loans.csv());
  }
}
```

`export.csv` مسار ثابت. ضعه قبل أي `@Get(':id')` على Express الافتراضي حتى لا يلتقطه المعامل، كما في فصل Controllers.

## ماذا تتأكد منه

- `@Global()` قبل الجذر في Middleware، و `app.use` قبلهما.
- خروج Interceptor عكس الدخول.
- المرشحات فقط عند استثناء غير ممسوك، ومن الدالة صعودًا.
- استثناء Middleware للمرشح العام فقط.
- `@Res()` بلا `passthrough` يكتب الرد خارج التسلسل القياسي.
