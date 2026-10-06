# Transactional Outbox

## الهدف من هذا الدرس

كتابة رسالة `project.published` في نفس معاملة نشر رف عبر `@nestjs/outbox`، بحيث لا تُرسل بلا صف ولا يُلتزم صف بلا رسالة.

## المشكلة التي يحلها

`UPDATE` نجح ثم سقطت العملية قبل `queue.add`. أو أُرسلت الرسالة ثم تراجعت المعاملة. إدراج يدوي في جدولكم يعمل، والحزمة تعطيه مخزنًا وترحيلًا ومعالجًا حتى لا تبنوا الناشر من الصفر.

## الفكرة ببساطة

`outbox.add` داخل `prisma.$transaction(async (tx) => ...)` مع تحديث المشروع. المخزن يرفض عميل Prisma الجذر لأن عباراته تُلتزم وحدها. مصفوفة `prisma.$transaction([...])` لا تمرر `tx` فلا تصلح. عامل الحزمة يسلّم لاحقًا و `@OnOutboxMessage` يعالج. التكرار ما زال ممكنًا إن سُلّمت الرسالة قبل تعليمها.

## التعريف التقني

`npm i @nestjs/outbox`. الصفحة تشرح المخزن على Postgres (`PostgresOutboxStore` من `@nestjs/outbox/postgres`) وتنص أن Prisma من بين العملاء: `fromPrisma(prisma)` والمنفذ الذي يُمرَّر إلى `add` هو عميل المعاملة التفاعلية. الجداول في schema اسمه `nest_outbox` ينشئه المخزن لا `schema.prisma`. في الإنتاج `npx nest-outbox migrate`. تمرير القاعدة نفسها يرمي `OutboxTransactionRequiredError`. `@OnOutboxMessage` للمستمع. هذا يكمّل لمحة الفصل 63 ولا يلغي مفتاح التكرار عند المستهلك.

## Mental Model

```text
$transaction(async tx => {
  update project
  outbox.add(..., tx)
})
relay لاحقًا → OnOutboxMessage
```

## أبسط مثال

```ts
await this.prisma.$transaction(async (tx) => {
  await tx.project.update({
    where: { id: projectId, organizationId },
    data: { status: 'published' },
  });
  await this.outbox.add(
    { type: 'project.published', payload: { projectId, organizationId } },
    tx,
  );
});
```

`Outbox` يُحقن من `@nestjs/outbox`. التوقيع الدقيق للرسالة في الصفحة؛ المهم تمرير `tx` لا `this.prisma`.

## شرح المثال

- التحديث مقيّد بالمنظمة.
- الرسالة حدث تكامل لا كيان Prisma.
- لا `add` بعد انتهاء المعاملة.

## ماذا يحدث داخليًا؟

1. المعاملة التفاعلية تفتح `tx`.
2. المشروع والرسالة يُكتبان على نفس الاتصال.
3. الالتزام معًا أو التراجع معًا.
4. المرحّل يقرأ `nest_outbox` ويدفع إلى الناقل أو إلى `@OnOutboxMessage`.
5. إن أُرسلت الرسالة وفشل التعليم تُعاد، والمستهلك يطبّق قيد المعرّف.

## مثال عملي

```ts
import { Injectable } from '@nestjs/common';
import { OnOutboxMessage } from '@nestjs/outbox';

@Injectable()
export class ProjectPublishedHandler {
  @OnOutboxMessage('project.published')
  handle(payload: { projectId: string; organizationId: string }) {
    return payload.projectId;
  }
}
```

المعالج يحمّل المشروع بـ `organizationId` من الحمولة. لا يثق بسياق طلب انتهى. لا يعيد `draft` عند فشل بريد لاحق.

## أخطاء شائعة

- `outbox.add(message)` بلا `tx`.
- مصفوفة استعلامات Prisma بدل callback.
- توقع أن `schema.prisma` يعرّف جداول `nest_outbox`.
- اعتبار التسليم إثبات أن الشريك أنهى عمله.

## متى نستخدمه؟

عندما يجب أن تُخلق رسالة التكامل إذا وفقط إذا نُشر المشروع.

## متى لا نستخدمه؟

لتدقيق داخل نفس الجداول أصلًا بلا ناقل. ولا كبديل عن `@Idempotent` على زر الإنشاء.

## مقارنة

| الأسلوب | بعد قتل العملية عقب الكتابة |
| --- | --- |
| `emit` ذاكرة | الرسالة تضيع |
| طابور بعد المعاملة | قد تضيع في النافذة |
| `outbox.add` مع `tx` | الصف يبقى ويُرحَّل |

## تمارين

### تمرين سهل

لماذا تُرفض `this.prisma` بدل `tx`؟

<details>
<summary>الحل</summary>

لأن عبارات العميل الجذر تُلتزم وحدها. الصفحة ترمي `OutboxTransactionRequiredError`. الرسالة قد تُرسل والنشر يتراجع.

</details>

### تمرين متوسط

لماذا لا تصلح `prisma.$transaction([update, insert])`؟

<details>
<summary>الحل</summary>

شكل المصفوفة لا يعطي عميل معاملة تفاعليًا لتمريره إلى `outbox.add`. استخدموا `async (tx) =>`.

</details>

### تمرين متقدم

المرحّل أرسل مرتين. ماذا يحمي الشريك؟

<details>
<summary>الحل</summary>

معرّف الرسالة الثابت وقيده عنده. الصندوق يمنع الضياع ولا يلغي التكرار بعد إرسال ناجح لم يُعلَّم.

</details>

## أسئلة مراجعة

- أي حزمة وأي عميل يُمرَّر إلى `add`؟
- أين تعيش جداول المخزن؟
- ماذا يرفض المخزن؟
- هل التسليم يعني أن الشريك أنهى؟

## الخلاصة

`@nestjs/outbox` يكتب الرسالة في معاملة Prisma التفاعلية عبر `fromPrisma`. المرحّل يسلّم لاحقًا. بلا `tx` لا ضمان. التكرار عند المستهلك يبقى لازمًا.

## ماذا بعد؟

بعض الأعمال تنتظر أيامًا وإشارة من ناقل، لا رسالة واحدة. الدرس التالي التدفقات المتينة.

## المصادر الرسمية

- NestJS — Transactional outbox: https://docs.nestjs.com/reliability/outbox
