# كلفة Request Scope

## الهدف من هذا الدرس

معرفة متى يُنشئ Nest نسخة جديدة من المزوّد لكل طلب، ولماذا تنتشر هذه الكلفة في الشجرة حتى لو وضعت `Scope.REQUEST` على صنف واحد في القاع.

## المشكلة التي يحلها

متجر رف يقرأ المستأجر أو المستخدم من الطلب، فيضع أحدهم `scope: Scope.REQUEST` على خدمة القاعدة «حتى يحمل كل طلب اتصاله». تحت الضغط يرتفع تخصيص الذاكرة وزمن الاستجابة، لأن المتحكم وكل من يحقن تلك الخدمة يُعاد بناؤه ثم يُجمع كقمامة بعد كل `GET /books`.

## الفكرة ببساطة

الافتراضي في Nest نسخة واحدة طول عمر التطبيق. هذا آمن على خيط JavaScript الواحد. النطاق `REQUEST` يقول: ابنِ النسخة مع الطلب وأتلفها بعده. أي مستهلك يحقن مزوّدًا بهذا النطاق يصير هو أيضًا لكل طلب.

## التعريف التقني

`Scope` من `@nestjs/common`: `DEFAULT` مشترك، `REQUEST` لكل طلب، `TRANSIENT` لكل مستهلك من غير أن يرفع نطاق المستهلك. حقن `REQUEST` من `@nestjs/core` يجعل المزوّد طلبيًا ولو لم تكتب النطاق. في GraphQL الرمز المقابل للطلب هو `CONTEXT`. التوثيق يذكر أن تطبيقًا مصممًا بعناية لا يُفترض أن يزيد التأخير أكثر من نحو 5٪، وأن الفقاعة هي الخطر: مزوّد مشترك في جذر الشجرة يحوّل أغلب التطبيق إلى نسخ سريعة الزوال. للقيم التي تتكرر بين طلبات (مستأجر، لا معرّف فريد لكل طلب) توجد مزودات durable مع `ContextIdFactory`. لقراءة المستخدم أو المستأجر مع إبقاء الجميع singleton، المخزن المناسب هو `AsyncLocalStorage`، وصفحة النطاقات تشير إليه.

## Mental Model

```text
BooksController  →  BooksService  →  TenantConnection (REQUEST)
       ↑ يُنشأ لكل طلب          ↑ يُنشأ لكل طلب
CatalogReader لا يحقن الاتصال          يبقى DEFAULT
```

`TRANSIENT` لا يصعد. `REQUEST` يصعد إلى من يحقنه.

## أبسط مثال

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import type { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class TenantContext {
  constructor(@Inject(REQUEST) private readonly request: Request) {}

  id() {
    const value = this.request.header('x-tenant-id');
    if (!value) throw new Error('missing tenant');
    return value;
  }
}
```

## شرح المثال

- كل طلب يحصل على `TenantContext` خاص به، لأن العنوان يتغير.
- `BooksService` إن حقنت `TenantContext` أصبحت طلبية ولو كان `@Injectable()` بلا خيار.
- `BooksController` يُعاد إنشاؤه لنفس السبب. هذا ثمن القراءة المباشرة للطلب عبر الحقن.

## ماذا يحدث داخليًا؟

1. عند الإقلاع تُبنى المزودات `DEFAULT` مرة واحدة.
2. يصل طلب. Nest ينشئ سياقًا (`contextId`) لذلك الطلب.
3. أي مزود `REQUEST` في سلسلة المتحكم يُبنى داخل هذا السياق.
4. بعد انتهاء الرد، النسخ الطلبية قابلة للجمع.
5. إن وُسم المزوّد `durable: true` مع استراتيجية `ContextIdStrategy`، تُعاد شجرة المستأجر نفسه بدل بناء شجرة لكل طلب. الاستراتيجية تُسجَّل بـ `ContextIdFactory.apply` قبل وصول الطلبات. التوثيق يحذّر أنها لا تناسب عددًا كبيرًا من المستأجرين.

## مثال عملي

قراءة المستأجر من غير هدم الشجرة. المخزن يُملأ في حارس، والخدمة تبقى singleton:

```ts
import { AsyncLocalStorage } from 'node:async_hooks';
import { Injectable } from '@nestjs/common';

export interface ShelfStore {
  tenantId: string;
}

export const shelfStorage = new AsyncLocalStorage<ShelfStore>();

@Injectable()
export class BooksService {
  constructor(private readonly prisma: { book: { findMany: Function } }) {}

  list() {
    const store = shelfStorage.getStore();
    if (!store) throw new Error('no tenant store');
    return this.prisma.book.findMany({
      where: { tenantId: store.tenantId },
    });
  }
}
```

الحارس يستدعي `shelfStorage.run({ tenantId }, () => next.handle())` عبر interceptor يغلّف التنفيذ، أو يدخل المتجر في بداية سلسلة تعرف كيف تبقي `run` محيطًا ببقية الـ async. الخدمة لا تُحقن بـ `REQUEST`. فصل المستأجرين يفصّل عقد هذا المخزن؛ هنا المقصد هو الكلفة.

## أخطاء شائعة

- وسم مزوّد القاعدة المركزي بـ `REQUEST` لأن حقلًا واحدًا يختلف بين الطلبات.
- توقع أن `TRANSIENT` يعزل الطلبات. هو يعزل المستهلكين داخل نفس العمر.
- وضع `Scope.REQUEST` على WebSocket gateway. البوابة يجب أن تبقى نسخة واحدة لأنها تمسك المقبس.
- قياس الإقلاع فقط. الكلفة تظهر لكل طلب، لا عند `bootstrap`.

## متى نستخدمه؟

عندما الحالة لا تُختصر إلى مفتاح تعيد استخدامه: ذاكرة تخزين مؤقت خاصة بالطلب في GraphQL، أو تتبع يموت مع الطلب، ولم يكفِ مخزن `AsyncLocalStorage`.

## متى لا نستخدمه؟

لقراءة `user` أو `tenantId` فقط. لذلك يوجد مخزن الطلب أو مزود durable إن كانت الشجرة تعتمد على مصدر بيانات لكل مستأجر وعدد المستأجرين صغير.

## مقارنة

| النطاق | عمر النسخة | هل يصعد؟ |
| --- | --- | --- |
| `DEFAULT` | عمر التطبيق | لا حاجة |
| `REQUEST` | طلب واحد | نعم، إلى من يحقنه |
| `TRANSIENT` | لكل مستهلك | لا |
| `REQUEST` + `durable` | شجرة تُعاد لمفتاح مشترك | حسب الاستراتيجية |

## تمارين

### تمرين سهل

`BooksService` بلا نطاق تحقن `AuditLogger` الموسوم `Scope.REQUEST`. ما عمر `BooksService` الفعلي؟

<details>
<summary>الحل</summary>

طلبي. الفقاعة تصعد من المسجّل إلى الخدمة ثم إلى أي متحكم يحقن الخدمة. `AuditLogger` لم يبقَ استثناءً معزولًا.

</details>

### تمرين متوسط

عشرة مستأجرين، ومصدر البيانات يعتمد على المستأجر لا على معرّف الطلب. لماذا يذكر التوثيق المزودات الدائمة هنا؟

<details>
<summary>الحل</summary>

لأن الطلبات تتجمع في عشر شجرات لا في شجرة لكل طلب. `ContextIdFactory.apply` يختار `contextId` المستأجر عندما تكون الشجرة durable. هذا لا يناسب آلاف المستأجرين، والتوثيق يقول ذلك.

</details>

### تمرين متقدم

ارسم شجرة `OrdersController → OrdersService → PaymentsClient (DEFAULT) → TenantContext (REQUEST)`. من يُنشأ لكل طلب؟

<details>
<summary>الحل</summary>

`TenantContext` و `OrdersService` و `OrdersController`. `PaymentsClient` يبقى singleton إذا لم يحقن `TenantContext`. إن حقنه أحدهم لاحقًا، يصعد النطاق إليه أيضًا.

</details>

## أسئلة مراجعة

- ما النطاق الافتراضي، ولماذا هو آمن في Node؟
- ماذا يفعل حقن رمز `REQUEST` حتى من غير كتابة `scope`؟
- متى تبقى الخدمة `DEFAULT` رغم اختلاف المستأجر بين الطلبات؟
- ما حد التأخير الذي يذكره توثيق النطاقات لتطبيق مصمم بعناية؟

## الخلاصة

`Scope.REQUEST` أداة عزل حقيقية وثمنها إعادة بناء الشجرة لكل طلب. اجعله قرارًا ضيقًا. لقراءة سياق الطلب، أبقِ الخدمات singleton وضع القيمة في مخزن يطلبه التنفيذ.

## ماذا بعد؟

الشجرة رخيصة وما زال `GET /books?author=نجيب` بطيئًا. الدرس التالي عن الاستعلام والفهرس لا عن الإطار.

## المصادر الرسمية

- NestJS — Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
- NestJS — Async local storage: https://docs.nestjs.com/recipes/async-local-storage
