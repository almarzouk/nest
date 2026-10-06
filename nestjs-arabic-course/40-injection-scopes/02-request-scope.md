# REQUEST

## الهدف من هذا الدرس

فهم نسخة لكل طلب، وحقن رمز `REQUEST`، وقاعدة العدوى: من يحقن مزود طلب يصبح نفسه نطاق طلب.

## المشكلة التي يحلها

أحيانًا تحتاج كائنًا يموت مع الرد: ذاكرة تخزين مؤقت لحل GraphQL واحد، أو مصدر بيانات يُختار من ترويسة المستأجر. الخلط الشائع هو تعليم الخدمة `REQUEST` فقط لتمرير `user`. ذلك يعيد بناء سلسلة الحقن لكل طلب. المستخدم يُقرأ من الطلب عبر Guard وديكور، والخدمة تبقى Singleton وتستقبل المعرف وسيطًا.

## الفكرة ببساطة

`Scope.REQUEST` ينشئ نسخة للطلب الوارد وتُجمع بعد معالجته. إن حقنها Controller أو Provider، يصعد النطاق في سلسلة الحقن. التابع الذي لا يحقنها يبقى `DEFAULT`. رمز `REQUEST` نفسه نطاق طلب بطبيعته، ومن يعتمد عليه يصعد حتى لو لم تكتب `scope`.

## التعريف التقني

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class TenantDb {}
```

على المتحكم:

```ts
import { Controller, Scope } from '@nestjs/common';

@Controller({ path: 'cats', scope: Scope.REQUEST })
export class CatsController {}
```

حقن الطلب في HTTP:

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import type { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class TenantDb {
  constructor(@Inject(REQUEST) private readonly request: Request) {}
}
```

كتابة `scope` مع `REQUEST` تُهمل لأن الرمز أصلًا نطاق طلب، والعدوى لا تُلغى. في GraphQL تحقن `CONTEXT` من `@nestjs/graphql` بدل `REQUEST`. بوابات WebSocket لا تستخدم هذا النطاق لأنها تغلف مقبسًا حقيقيًا ويجب أن تبقى Singleton، ومثلها بعض استراتيجيات Passport ومتحكمات Cron.

رسم التوثيق: `CatsController <- CatsService <- CatsRepository`. إن كانت `CatsService` نطاق طلب، يصبح المتحكم نطاق طلب. `CatsRepository` إن لم يحقن الخدمة يبقى Singleton.

خطافات دورة حياة التطبيق لا تُستدعى على هذه الأصناف.

## Mental Model

```text
طلب وارد -> context id -> نسخة TenantDb ونسخة كل من يحقنها
الرد خرج -> النسخ تُجمع
Repository لم يحقنها -> النسخة القديمة Singleton
```

العدوى تصعد إلى المستهلك، ولا تنزل إلى من لا يعتمد على المزود.

## أبسط مثال

```ts
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest<{ user?: { id: string } }>();
    return request.user;
  },
);
```

Guard وضع `request.user` قبل المعالج. الديكور يقرأ الحقل. `BooksService.findFor(userId)` تبقى `Scope.DEFAULT`. لا `REQUEST` في السلسلة.

## شرح المثال

هذا بديل تمرير المستخدم، وهو المسار الذي يقصده التحذير: لا تغيّر عمر الخدمة لتقرأ قيمة موجودة على كائن الطلب. الديكور يعمل عند المعالج فقط. الخدمة لا تعرف Express إن استقبلت `string`.

## ماذا يحدث داخليًا؟

1. يصل طلب فيُنشأ context id.
2. أي مزود `REQUEST` في سلسلة هذا المتحكم يُبنى داخل تلك الشجرة الفرعية.
3. المتحكم الذي يحقن المزود يُبنى هو أيضًا لهذا الطلب ثم يُجمع.
4. مزود `DEFAULT` خارج السلسلة يُعاد من الكاش.
5. بعد الرد تُجمع النسخ المؤقتة. لا `onModuleDestroy` لها.
6. في GraphQL السياق يحمل الطلب إن هيأت `context` في `GraphQLModule`.

## مثال عملي

مصدر بيانات يختلف حقًا لكل طلب لأن الاتصال يُختار من ترويسة:

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import type { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class TenantDb {
  constructor(@Inject(REQUEST) request: Request) {
    this.schema = String(request.headers['x-tenant-id'] ?? 'public');
  }

  readonly schema: string;
}
```

كل من يحقن `TenantDb` يصبح نطاق طلب. إن حقنته خدمة كتب واسعة، أعدت بناء تلك الخدمة لكل طلب. الدرس قبل الأخير في الفصل يحسب هذه الكلفة، والدرس الأخير يشرح `durable` عندما يتجمع الطلبات على مستأجر لا على طلب.

## أخطاء شائعة

- تعليم كل الخدمات `REQUEST` لتمرير المستخدم.
- نسيان أن العدوى تصعد إلى المتحكم.
- توقع `onModuleInit` على الصنف.
- وضع النطاق على Gateway.
- افتراض أن كتابة `Scope.DEFAULT` على المستهلك تلغي العدوى. التوثيق يقول إن السلوك لا يُلغى.

## متى نستخدمه؟

نسخة يجب أن تموت مع الطلب: كاش لحل واحد، أو تتبع يخص الطلب، أو مصدر يقرأ الطلب نفسه ولا يمكن تجميعه في شجرة أطول عمرًا.

## متى لا نستخدمه؟

لقراءة المستخدم أو اللغة أو المستأجر إن كانت القيمة وسيطًا أو حقلًا على الطلب. Guard يملأ الحقل، والديكور أو وسيط الدالة يوصله، والخدمة Singleton.

## مقارنة

| الحاجة | الآلية |
| --- | --- |
| المستخدم الحالي | Guard ثم `@CurrentUser()` ووسيط |
| كائن يُبنى من الطلب ويموت معه | `Scope.REQUEST` |
| نسخة لكل مكان حقن داخل الطلب نفسه | `TRANSIENT` |

## تمارين

### تمرين سهل

من يصعد إلى نطاق الطلب: الذي يحقن المزود أم الذي يُحقن هو؟

<details>
<summary>الحل</summary>

المستهلك الذي يحقن مزود `REQUEST` يصعد. المزود الذي لا يحقن أحدًا من السلسلة يبقى على نطاقه.

</details>

### تمرين متوسط

لماذا يُهمل `scope` إن حقنت `REQUEST`؟

<details>
<summary>الحل</summary>

لأن الرمز نطاق طلب أصلًا، وأي معتمد عليه يتبنى النطاق، وهذا لا يُغيَّر.

</details>

### تمرين متقدم

`BooksService` حقنت `TenantDb` نطاق طلب، و `BooksService` موثقة كـ Singleton في ذهنك. ماذا يصبح المتحكم الذي يحقن `BooksService`؟

<details>
<summary>الحل</summary>

`BooksService` تصعد إلى `REQUEST` بالعدوى، والمتحكم يحقنها فيصعد أيضًا. تُبنى نسخة متحكم لكل طلب.

</details>

## أسئلة مراجعة

- متى تُجمع النسخة؟
- ماذا تحقن في GraphQL بدل `REQUEST`؟
- لماذا الديكور أصلح لتمرير المستخدم؟

## الخلاصة

`REQUEST` نسخة للطلب وتعدي صعودًا. استعملها عندما تختلف النسخة نفسها، واقرأ المستخدم من الطلب من غير تغيير عمر الخدمة.

## ماذا بعد؟

`TRANSIENT`: نسخة لكل مستهلك، من غير أن يصعد النطاق.

## المصادر الرسمية

- NestJS — Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
