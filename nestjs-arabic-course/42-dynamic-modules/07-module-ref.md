# ModuleRef

## الهدف من هذا الدرس

التنقل في قائمة المزودات وقت التشغيل بـ `ModuleRef`: `get` للنسخة المسجّلة، و `resolve` لنطاق طلب أو عابر، و `create` لصنف لم يُسجَّل.

## المشكلة التي يحلها

الحقن في الـ `constructor` يكفي لمعظم الخدمات. تحتاج الرمز لاحقًا عندما تفك طرف دورة، أو تختار صنفًا حسب شرط بعد الإقلاع، أو تبني مزودًا عابرًا أكثر من مرة. `new` المباشر يتخطى الحاوية. `ModuleRef` من `@nestjs/core` يبحث بالرمز داخلها.

## الفكرة ببساطة

احقن `ModuleRef`. `get(Token)` يعيد Singleton (أو متحكمًا أو حارسًا) مسجّلًا في الوحدة الحالية، ويرمي إن غاب. `{ strict: false }` يوسع البحث إلى السياق العام. `resolve(Token)` يبني شجرة فرعية فريدة للمزود ذي النطاق، و `ContextIdFactory` يشارك تلك الشجرة بين نداءات `resolve`. `create(Class)` يبني صنفًا لم يُسجل كمزود ويحقن ما يستطيع.

## التعريف التقني

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';
import { Service } from './service.js';

@Injectable()
export class CatsService implements OnModuleInit {
  private service!: Service;

  constructor(private readonly moduleRef: ModuleRef) {}

  onModuleInit(): void {
    this.service = this.moduleRef.get(Service);
  }
}
```

خارج الوحدة: `this.moduleRef.get(Service, { strict: false })`.

العابر:

```ts
const a = await this.moduleRef.resolve(TransientService);
const b = await this.moduleRef.resolve(TransientService);
```

`a === b` يعطي `false`. لجعلهما متساويين:

```ts
import { ContextIdFactory } from '@nestjs/core';

const contextId = ContextIdFactory.create();
const a2 = await this.moduleRef.resolve(TransientService, contextId);
const b2 = await this.moduleRef.resolve(TransientService, contextId);
```

معرف تصنعه أنت يجعل `REQUEST` داخل تلك الشجرة `undefined` حتى تسجّل طلبًا: `moduleRef.registerRequestByContextId(request, contextId)`.

لشجرة الطلب الحالي: حقن `REQUEST` ثم `ContextIdFactory.getByRequest(this.request)` ومرر الناتج إلى `resolve`. هذا يلزمك أصلًا بنطاق طلب، مع كلفته.

`create`:

```ts
this.catsFactory = await this.moduleRef.create(CatsFactory);
```

الصنف لم يكن في `providers`. المفيد عندما تختار تنفيذًا بشرط. لا تستعمله بديلًا لتسجيل الخدمة العادية.

## Mental Model

```text
get      -> نسخة موجودة، DEFAULT، strict للوحدة
resolve  -> شجرة جديدة لكل نداء، أو شجرة contextId
create   -> صنف خارج التسجيل
```

`get` على `REQUEST` أو `TRANSIENT` ليس المسار الصحيح. التوثيق يحيل إلى `resolve`.

## أبسط مثال

مقطع `onModuleInit` مع `get`. ضعه على الطرف الذي كان يسبب دورة: لا تحقن `OrdersService` في البناء، واجلبها بعد حل الوحدة. `{ strict: false }` إن كانت مسجّلة في وحدة أخرى مرئية.

## شرح المثال

`OnModuleInit` بعد حل اعتماديات الوحدة، فالتسجيل الساكن جاهز. الحقل `service` يُستخدم في الدوال بعدها. لا تنادِ `get` في `constructor`. إن رمى `get` فالرموز غير مسجّل في هذا النطاق، لا أن الحاوية معطلة.

## ماذا يحدث داخليًا؟

1. `ModuleRef` المرتبط بالصنف يرى سجل وحدته.
2. `get` يبحث بالمطابقة الافتراضية الصارمة.
3. `strict: false` يصعد إلى المزودات العامة والمصدَّرة في السياق الأوسع.
4. `resolve` ينشئ context id إن لم تمرره، ويبني الفرع، ويعيد نسخة ذلك الفرع.
5. تكرار `resolve` بلا المعرف نفسه يعيد كائنًا آخر.
6. `create` يسلك الحقن لصنف خارج القائمة ثم يعيد النسخة لك، بلا تسجيل دائم كمزود وحدة.

## مثال عملي

معالج ملفات يختار `PdfRenderer` أو `HtmlRenderer` حسب اللاحقة، والصنفان غير مسجلين معًا دائمًا:

```ts
import { Injectable } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';
import { PdfRenderer } from './pdf.renderer.js';

@Injectable()
export class RenderPicker {
  constructor(private readonly moduleRef: ModuleRef) {}

  async pdf(): Promise<PdfRenderer> {
    return this.moduleRef.create(PdfRenderer);
  }
}
```

إن كان `PdfRenderer` يحتاج `PrismaService` مسجّلة، `create` يحقنها من الحاوية. إن كان يحتاج رمزًا غير موجود، يفشل البناء كما يفشل الإقلاع العادي. لتكرار الاستعمال سجّل الصنف في `providers` واحقنه، واترك `create` للقرار الشرطي.

## أخطاء شائعة

- `get` لمزود `REQUEST` وتوقع طلب المستخدم.
- `get` في `constructor`.
- نسيان `strict: false` لمزود من وحدة أخرى ثم ابتلاع الاستثناء بـ `forwardRef` جديد.
- `resolve` في كل طلب من Singleton فيصنع أشجارًا بلا قياس.
- `create` لكل نداء على صنف ثقيل بدل تسجيله مرة.

## متى نستخدمه؟

جلب متأخر لـ Singleton، عزل عابر بـ `resolve`، أو بناء شرطي بـ `create`.

## متى لا نستخدمه؟

بدل حقن عادي تعرفه عند الإقلاع. الحاوية عندها تفشل باكرًا إن نقص رمز، وهذا أفضل من `get` اختياري يخفي الخطأ حتى أول طلب.

## مقارنة

| الدالة | التسجيل مسبقًا | النطاق |
| --- | --- | --- |
| `get` | نعم | Singleton في الوحدة، أو أوسع مع `strict: false` |
| `resolve` | نعم | طلب أو عابر، شجرة لكل context |
| `create` | لا | يبني صنفًا ويحقن ما يجده |

## تمارين

### تمرين سهل

من أي حزمة تأتي `ModuleRef`؟

<details>
<summary>الحل</summary>

`@nestjs/core`.

</details>

### تمرين متوسط

نداءان `resolve(TransientService)` بلا `contextId`. هل المرجعان متساويان؟

<details>
<summary>الحل</summary>

لا. كل نداء شجرة فرعية فريدة.

</details>

### تمرين متقدم

كيف تشارك `resolve` مع طلب HTTP الحالي؟

<details>
<summary>الحل</summary>

احقن `REQUEST`، خذ المعرف بـ `ContextIdFactory.getByRequest(this.request)`، ومرره إلى `resolve`. الصنف الذي يحقن `REQUEST` يصبح نطاق طلب بالعدوى.

</details>

## أسئلة مراجعة

- متى `strict: false`؟
- لماذا `get` لا يناسب العابر؟
- ماذا لا يسجّله `create` تلقائيًا كمزود دائم؟

## الخلاصة

`ModuleRef` بوابة الحاوية بعد البناء: `get` للموجود، `resolve` لذي النطاق، `create` لصنف خارج القائمة. الحقن العادي يبقى الطريق الأول.

## ماذا بعد؟

Lazy-loading Modules: تحميل وحدة عند الحاجة، وحدود ما لا يُحمَّل متأخرًا.

## المصادر الرسمية

- NestJS — Module reference: https://docs.nestjs.com/fundamentals/module-ref
