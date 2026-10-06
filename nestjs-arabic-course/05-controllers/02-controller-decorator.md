# @Controller

## الهدف من هذا الدرس

قراءة `@Controller` كإعلان بادئة للمجموعة، ومعرفة شكل الوسيط النصي وشكل الكائن.

## المشكلة التي يحلها

تكرار `/invoices` في كل دالة يفرق المسارات عند إعادة التسمية. بادئة واحدة على الصنف تبقي المجموعة معًا، وتقصير كل Decorator على الدالة إلى الجزء المتبقي.

## الفكرة ببساطة

`@Controller('invoices')` يلزم Nest بإنشاء Controller ويمنحه بادئة. المسار الكامل هو البادئة ثم المسار المكتوب على دالة الطريقة.

## التعريف التقني

الوسيطة اختيارية. `@Controller()` بلا نص يضع الدوال على جذر التطبيق، ما لم تُضف بادئة عامة بـ `setGlobalPrefix`.

الشكل النصي يساوي `path` داخل الكائن:

```ts
@Controller('invoices')
@Controller({ path: 'invoices' })
```

الكائن يقبل أيضًا `host` حتى يطابق اسم المضيف. Fastify لا يدعم المسارات الفرعية المتداخلة التي يحتاجها host routing؛ مع `host` استخدم محول Express الافتراضي.

رمز في المضيف يُقرأ بـ `@HostParam()`:

```ts
@Controller({ host: ':tenant.example.com', path: 'invoices' })
```

خيار `scope` على الكائن يغيّر عمر النسخة. شرح Scope في فصل Providers. اتركه فارغًا هنا حتى تبقى النسخة Singleton.

## Mental Model

```text
@Controller('invoices') + @Get('overdue')
  = GET /invoices/overdue

@Controller('invoices') + @Post()
  = POST /invoices
```

البادئة تجميع، وهي تقلل تكرار النص. هي جزء من عنوان المورد.

## أبسط مثال

```ts
import { Controller, Get, Post } from '@nestjs/common';
import { InvoicesService } from './invoices.service.js';

@Controller('invoices')
export class InvoicesController {
  constructor(private readonly invoices: InvoicesService) {}

  @Get('overdue')
  overdue() {
    return this.invoices.findOverdue();
  }

  @Post()
  open() {
    return this.invoices.openDraft();
  }
}
```

## شرح المثال

- `overdue` نص إضافي يُضاف بعد البادئة، فيصبح `GET /invoices/overdue`.
- `@Post()` بلا نص يعني `POST /invoices`، أي مورد المجموعة نفسها.
- الصنف ما زال يحتاج تسجيلًا في `controllers`. الـ Decorator يصف المسار بعد أن تُنشأ النسخة.
- الدالتان تعيدان قيمة الخدمة. رمز POST الافتراضي 201، ويُفصَّل مع `@HttpCode`.

## ماذا يحدث داخليًا؟

Nest تخزن `path` في Metadata الصنف. عند بناء الخريطة تدمج بادئة Controller مع مسار كل طريقة. اسم المضيف، إن وُجد، قيد إضافي قبل مطابقة المسار: طلب `api.example.com` لا يدخل Controller مشروطًا بـ `admin.example.com`.

`@HostParam('tenant')` يقرأ الجزء الملتقط من المضيف، بالطريقة التي يقرأ بها `@Param` جزءًا من المسار. درس Route Params يبقى لمسار URL.

## مثال عملي

```ts
import { Controller, Get, HostParam } from '@nestjs/common';

@Controller({ host: ':tenant.example.com', path: 'invoices' })
export class TenantInvoicesController {
  @Get('open')
  open(@HostParam('tenant') tenant: string) {
    return { tenant, view: 'open-invoices' };
  }
}
```

طلب `GET https://north.example.com/invoices/open` يملأ `tenant` بالقيمة `north`. بادئة التطبيق العامة تُضاف قبل `invoices` إن استدعيت `app.setGlobalPrefix('api')`، فيصبح المسار `/api/invoices/open` على المضيف نفسه.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| كتابة `/invoices` داخل كل `@Get` ونسيان البادئة على الصنف | التكرار يتفرع عند تغيير الاسم |
| بادئة `invoices` على الصنف ومسار `invoices/overdue` على الدالة | المسار يتضاعف إلى `/invoices/invoices/overdue` |
| `host` مخصص مع Fastify | التوثيق يطلب Express لهذا الخيار |
| صنف بلا `@Controller()` داخل `controllers` | لا بادئة ولا خريطة |

## متى نستخدمه؟

على كل صنف يمثل مجموعة REST واحدة. النص القصير هو الشكل المعتاد. الكائن عندما تحتاج `host` مع `path`.

## متى لا نستخدمه؟

لتصفية `?status=overdue`. ذلك Query Param على دالة `@Get()`. البادئة تثبت جزء المسار الثابت للمجموعة.

## مقارنة

| الوسيط | متى يظهر في العنوان |
| --- | --- |
| `'invoices'` | بادئة مسار لكل دوال الصنف |
| `{ path, host }` | البادئة زائد قيد اسم المضيف |
| نص داخل `@Get('overdue')` | قطعة تُلحَق بالبادئة لتلك الدالة فقط |

## تمارين

### تمرين سهل

ما مسار `@Controller('invoices')` مع `@Get('overdue')`؟

<details><summary>الحل</summary>

`GET /invoices/overdue`.

</details>

### تمرين متوسط

ما الفرق بين `@Controller('invoices')` و `@Controller({ path: 'invoices' })`؟

<details><summary>الحل</summary>

البادئة واحدة. شكل الكائن هو الذي يسمح بإضافة `host` أو `scope` في الوسيط نفسه.

</details>

### تمرين متقدم

لماذا `@Get('invoices/overdue')` مع بادئة `invoices` ينتج مسارًا غير الذي قصده الفريق؟

<details><summary>الحل</summary>

الدمج يلصق الاثنين. الناتج `/invoices/invoices/overdue`. القطعة على الدالة هي ما بعد البادئة: `@Get('overdue')`.

</details>

## أسئلة مراجعة

- ماذا تفعل وسيطة `@Controller` النصية؟
- متى يُستخدم شكل الكائن؟
- من أين يُقرأ رمز المضيف الديناميكي؟

## الخلاصة

`@Controller` يثبت بادئة المجموعة ويمكن أن يقيد المضيف. مسار الدالة يُلحَق بهذه البادئة، والصنف يبقى مسجّلًا في Module.

## ماذا بعد؟

كيف تُركَّب القطعة وتُحسم التعارضات: Routing.

## المصادر الرسمية

- NestJS — Controllers (Routing): https://docs.nestjs.com/controllers#routing
