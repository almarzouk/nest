# Handlers

## الهدف من هذا الدرس

تنفيذ الأمر في `@CommandHandler` والاستعلام في `@QueryHandler`، وتسجيل الصنف كمزود حتى يكتشفه `CqrsModule`.

## المشكلة التي يحلها

كائن الأمر بلا معالج لا يغيّر حالة. وضع القاعدة في المتحكم أو في صنف الأمر يعيد الخلط. المعالج مكان `execute`: يجلب الجذر، ينادي دواله، ويحفظ عبر المستودع. الاستعلام معالج آخر لا يستدعي `save`.

## الفكرة ببساطة

`@CommandHandler(CancelOrderCommand)` على صنف يطبّق `ICommandHandler<CancelOrderCommand>` وفيه `execute(command)`. `@QueryHandler(GetOrderQuery)` كذلك مع `IQueryHandler`. الصنف في `providers`. `CqrsModule.forRoot()` في الوحدة الجذر يفعّل الاكتشاف. المتحكم يحقن `CommandBus` وينادي `execute(new CancelOrderCommand(id))`.

## التعريف التقني

```ts
import { CommandHandler, type ICommandHandler } from '@nestjs/cqrs';
import { CancelOrderCommand } from './cancel-order.command.js';
import { ORDERS, type OrdersRepository } from '../orders/orders.repository.js';
import { Inject } from '@nestjs/common';

@CommandHandler(CancelOrderCommand)
export class CancelOrderHandler implements ICommandHandler<CancelOrderCommand> {
  constructor(@Inject(ORDERS) private readonly orders: OrdersRepository) {}

  async execute(command: CancelOrderCommand): Promise<void> {
    const order = await this.orders.findById(command.orderId);
    if (order === null) {
      throw new Error('الطلب غير موجود');
    }
    order.cancel();
    await this.orders.save(order);
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';

@Module({
  imports: [CqrsModule.forRoot()],
})
export class AppModule {}
```

`forRoot()` مرة. الخيارات مثل ناشر الأوامر ونوع إعادة رمي الاستثناءات غير المعالجة موثقة في الوصفة. الافتراض يكفي حتى تحتاج `rethrowUnhandled`.

```ts
constructor(private readonly commandBus: CommandBus) {}

cancel(id: string) {
  return this.commandBus.execute(new CancelOrderCommand(id));
}
```

`execute` على الحافلة يعيد وعد ناتج المعالج. معالج الاستعلام:

```ts
@QueryHandler(GetOrderQuery)
export class GetOrderHandler implements IQueryHandler<GetOrderQuery> {
  async execute(query: GetOrderQuery) {
    return this.reader.view(query.orderId);
  }
}
```

`reader` قارئ نموذج العرض لا `OrdersRepository.save`.

الحافلات Singleton. إن احتجت مزود طلب داخل المعالج، الوصفة تطلب وسم النطاق على المزخرف نفسه مثل `@CommandHandler(CancelOrderCommand, { scope: Scope.REQUEST })` لأن الحافلة لا تصبح نطاق طلب تلقائيًا بالطريقة الساذجة. تجنّب ذلك لأجل المستخدم. مرر `userId` داخل الأمر.

سجّل كل معالج في `providers` لوحدته. الاستيراد `CqrsModule` من غير أن يكون المعالج مزودًا يترك الأمر بلا معالج.

## Mental Model

```text
Controller -> CommandBus.execute(command)
           -> CancelOrderHandler.execute
           -> OrdersRepository
QueryBus   -> GetOrderHandler -> قارئ العرض
```

## أبسط مثال

المقاطع أعلاه. اختبار المعالج بـ `new CancelOrderHandler(memory)` ينادي `execute` بلا حافلة. اختبار الربط يستعمل `TestingModule` مع `CqrsModule` والمعالج ويتأكد أن الحافلة تصل إليه.

## شرح المثال

`execute` هو دالة الخدمة السابقة باسم ثابت تتوقعه الحزمة. المستودع نفسه من فصل الدومين. CQRS لم يستبدل Prisma ولم يستبدل الجذر. أضاف توجيهًا من نوع الأمر إلى صنف.

## ماذا يحدث داخليًا؟

1. الإقلاع يستكشف المزودات ذات المزخرف.
2. يربط `CancelOrderCommand` بالمعالج.
3. `commandBus.execute` يجد الربط ويستدعي `execute`.
4. الاستثناء يصعد للمستدعي في أوامر المتحكم عادة.
5. معالج الأحداث مسار مختلف: الاستثناء غير الملتقط يذهب إلى `UnhandledExceptionBus` إلا إذا `rethrowUnhandled: true`.
6. أمر بلا معالج فشل وقت النداء. لا مسار صامت.

## مثال عملي

`IssueInvoiceHandler` يحقن `ORDERS` و `INVOICES`. يقرأ الطلب المفتوح، يصدر الفاتورة، يحفظ التجميعين. `GetInvoiceViewHandler` يحقن قارئ SQL أو Prisma يعيد DTO اللوحة ولا يحقن `save`. وحدتان أو وحدة فوترة تسجل المعالجين وتستورد `CqrsModule` إن لم يكن `forRoot` عام الأثر عبر الجذر. اتبع تصدير الحزمة: `forRoot` في الجذر يكفي لرؤية المعالجات المسجّلة.

## أخطاء شائعة

- نسيان المعالج في `providers`.
- `forRoot()` في كل ميزة مع إعداد متعارض.
- استعلام يستدعي `save`.
- `Scope.REQUEST` على المعالج لتمرير المستخدم.
- اختبار عبر HTTP فقط فلا يفشل معالج بلا تسجيل إلا متأخرًا.

## متى نستخدمه؟

بعد أن صار للأمر والاستعلام معنيان مختلفان وتريد توجيه الحزمة.

## متى لا نستخدمه؟

دالة `BooksService.create` كافية. المعالج الذي يستدعي الخدمة القديمة بلا فرق يعيد تسمية الطبقات.

## مقارنة

| | Command handler | Query handler |
| --- | --- | --- |
| المزخرف | `@CommandHandler` | `@QueryHandler` |
| الواجهة | `ICommandHandler` | `IQueryHandler` |
| الدالة | `execute` | `execute` |
| التخزين | كتابة عبر المستودع | قراءة عبر نموذج العرض |

## تمارين

### تمرين سهل

أين يُسجَّل `CancelOrderHandler`؟

<details>
<summary>الحل</summary>

في `providers` لوحدته، مع `CqrsModule.forRoot()` في جذر التطبيق.

</details>

### تمرين متوسط

كيف تختبر القاعدة بلا `CommandBus`؟

<details>
<summary>الحل</summary>

`new CancelOrderHandler(inMemoryOrders())` ثم `execute(new CancelOrderCommand('o1'))`.

</details>

### تمرين متقدم

لماذا تمرير `userId` في الأمر أفضل من `@CommandHandler(..., { scope: Scope.REQUEST })`؟

<details>
<summary>الحل</summary>

الحافلات Singleton، ونطاق الطلب على المعالج يعيد بناءه لكل أمر ويسحب كلفة النطاق. الحقل في الأمر يبقي المعالج الافتراضي.

</details>

## أسئلة مراجعة

- ماذا يطابق الحافلة بالمعالج؟
- ماذا يفعل استعلام يستدعي `save`؟
- ما دور `forRoot`؟

## الخلاصة

المعالج `execute` المسجّل كمزود هو مكان القاعدة. الأمر يُوجَّه بـ `CommandBus` والاستعلام بـ `QueryBus`. اختبر المعالج مباشرة، واترك نطاق الطلب خارج الحافلة.

## ماذا بعد؟

Event و EventBus و CommandBus و QueryBus: النشر بعد التغيير وكيف تتصل الحافلات.

## المصادر الرسمية

- NestJS — CQRS: https://docs.nestjs.com/recipes/cqrs
