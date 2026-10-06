# Event و EventBus و CommandBus و QueryBus

## الهدف من هذا الدرس

تمييز الحافلات الثلاث: أمر يُنفذ ويُنتظر، استعلام يُقرأ ويُنتظر، وحدث يُنشر بعد التغيير فيعالج `@EventsHandler` من غير أن يعيد ناتجًا للمسار.

## المشكلة التي يحلها

`CancelOrderHandler` الذي يستدعي البريد والمخزون واللوحة مباشرة يعود كيس آثار. الحدث `OrderCancelled` يخبر المشتركين بعد نجاح الحفظ. `CommandBus` و `QueryBus` يبقيان مدخل المسار لأنهما يعيدان ناتجًا. `EventBus` ناقل داخل العملية في هذه الحزمة، لا ضمان شبكة ولا مخزن أحداث إلا إذا بنيته.

## الفكرة ببساطة

بعد تغيير الجذر استدعِ `apply` على `AggregateRoot` ثم اربط النموذج بـ `EventPublisher.mergeObjectContext` ثم `commit()` فينشر الأحداث المعلقة. أو `eventBus.publish(event)` مباشرة. `@EventsHandler(OrderCancelled)` على مزود فيه `handle`. المعالج حدث لا يفشل طلب HTTP بالضرورة: الاستثناء غير الملتقط يذهب إلى `UnhandledExceptionBus` ما لم تضع `rethrowUnhandled: true` في `forRoot`.

## التعريف التقني

حافلات تُحقن من `@nestjs/cqrs`:

| الحافلة | النداء | الناتج |
| --- | --- | --- |
| `CommandBus` | `execute(command)` | وعد ناتج المعالج |
| `QueryBus` | `execute(query)` | وعد نموذج القراءة |
| `EventBus` | `publish(event)` | معالجة لاحقة داخل العملية |

```ts
export class OrderCancelled {
  constructor(public readonly orderId: string) {}
}
```

```ts
@EventsHandler(OrderCancelled)
export class OrderCancelledHandler implements IEventHandler<OrderCancelled> {
  handle(event: OrderCancelled): void {
    void event.orderId;
  }
}
```

من الوصفة، الجذر يمتد `AggregateRoot` ويستدعي `apply(new HeroKilledDragonEvent(...))` داخل السلوك. المعالج يدمج السياق:

```ts
const order = this.publisher.mergeObjectContext(await this.orders.findById(id));
order.cancel();
await this.orders.save(order);
order.commit();
```

`apply` يجمّع حتى `commit`. من غير `mergeObjectContext` لا يعرف الجذر الحافلة. `@nestjs/cqrs` 12.1 يمرر سياق موزع عبر `publish` و `publishAll` عند الدمج. إن كان الكيان لا يستطيع الامتداد، `WithAggregateRoot` أو `IAggregateRoot` كما في الوصفة. للبقاء بلا إطار داخل الكيان، انشر الحدث من المعالج بعد `save` بـ `eventBus.publish` وأبقِ `Order` حرًا من `AggregateRoot`. ذلك أنسب لمشروع صغير يريد حدثًا واحدًا.

`@Saga()` دالة ترجع `Observable` من أحداث إلى أوامر عبر `ofType`. ساجا متينة ومسار عمل في إصدارات حديثة من الحزمة عندما تمتد العملية زمنًا. لا تبدأ بها قبل معالج حدث واحد واضح.

الحافلات `Observable`. الاشتراك في `EventBus` يصلح لسجل تشخيصي. ليس بديل مستودع.

## Mental Model

```text
CommandBus.execute -> Handler -> save -> commit/publish
                                      -> EventBus
                                      -> EventsHandler (بريد، قراءة)
QueryBus.execute  -> Handler قراءة فقط
```

الحدث بعد الحفظ الناجح. نشره قبل `save` يخبر الناس بتغيير لم يُكتب.

## أبسط مثال

```ts
constructor(private readonly eventBus: EventBus) {}

await this.orders.save(order);
this.eventBus.publish(new OrderCancelled(order.snapshot().id));
```

`OrderCancelledHandler` يحدّث جدول قراءة أو يضع رسالة في الطابور. المتحكم حصل على ناتج الأمر قبل أن ينهي كل المعالجات إن كانت المعالجة غير متزامنة. لا تعتمد على البريد في جسم الاستجابة.

## شرح المثال

النشر اليدوي من المعالج يبقي الكيان بلا قاعدية الحزمة. `commit` أنسب عندما تتعدد `apply` داخل دوال الجذر وتريد إرسالها دفعة بعد النجاح. اختر أسلوبًا واحدًا في الوحدة حتى لا يُنشر الحدث مرتين.

## ماذا يحدث داخليًا؟

1. `CommandBus` يجد معالج الأمر وينتظره.
2. بعد `save` يُنشر الحدث على `EventBus`.
3. معالجات `@EventsHandler` المسجّلة تُستدعى.
4. فشلها يُسجل ويصل `UnhandledExceptionBus` إلا مع `rethrowUnhandled`.
5. `QueryBus` لا يرى الحدث إلا إذا حدّث المعالج نموذجًا يقرأه الاستعلام لاحقًا.
6. لا عبور شبكة. خدمة ثانية في عملية أخرى ليست مشتركة في هذه الحافلة.

## مثال عملي

إلغاء الطلب ينشر `OrderCancelled`. معالج يكتب صف `order_view` للاستعلام `GetOrderQuery`. معالج آخر يضع إشعارًا. `GetOrderQuery` لا يحقن مستودع الكتابة. إن فشل تحديث العرض، راقب `UnhandledExceptionBus` وأعد المحاولة من جدول، لأن إعادة رمي الخطأ لا تعيد الأمر تلقائيًا. ساجا تحول الحدث إلى `RestockCommand` فقط عندما تكون سلسلة الأوامر قاعدة عمل لا أثرًا جانبيًا واحدًا.

## أخطاء شائعة

- نشر قبل `save`.
- توقع جسم HTTP بعد اكتمال البريد.
- `QueryBus` لتعديل حالة.
- معالج حدث يحقن `CommandBus` في حلقة مع الأمر الذي نشر الحدث.
- اعتبار `EventBus` طابور Rabbit. ذلك فصل الخدمات الصغيرة.

## متى نستخدمه؟

آثار متعددة بعد كتابة ناجحة، أو تحديث نموذج قراءة داخل العملية.

## متى لا نستخدمه؟

أثر واحد سطر بريد داخل المعالج أوضح من حدث ومعالج. ناتج مطلوب في نفس الاستجابة يُحسب قبل `return` لا عبر حدث.

## مقارنة

| الحافلة | ينتظر المستدعي العمل؟ | يغيّر حالة؟ |
| --- | --- | --- |
| `CommandBus` | نعم | عبر معالج الأمر |
| `QueryBus` | نعم | لا |
| `EventBus` | لا تبنِ الرد عليه | عبر معالجات الحدث |

## تمارين

### تمرين سهل

أي حافلة تنشر `OrderCancelled`؟

<details>
<summary>الحل</summary>

`EventBus.publish`، أو `commit` بعد `apply` على جذر مرتبط بـ `EventPublisher`.

</details>

### تمرين متوسط

لماذا `UnhandledExceptionBus` يهم معالج الحدث أكثر من معالج الأمر في المتحكم؟

<details>
<summary>الحل</summary>

معالج الأمر يرفع الاستثناء إلى `execute` الذي ينتظره المتحكم. معالج الحدث يعمل بشكل لا يُرجع للنتيجة، والاستثناء غير الملتقط يُنشر على حافلة الاستثناءات إلا إذا فعّلت `rethrowUnhandled`.

</details>

### تمرين متقدم

متى تختار `eventBus.publish` من المعالج بدل `AggregateRoot`؟

<details>
<summary>الحل</summary>

عندما تريد الإبقاء على `Order` بلا قاعدة الحزمة ونشر حدث واحد بعد `save`. `apply` و `commit` عندما تتعدد الأحداث داخل دوال الجذر ويجب إرسالها بعد النجاح دفعة واحدة.

</details>

## أسئلة مراجعة

- ماذا يعيد `CommandBus.execute`؟
- متى يُنشر الحدث نسبة إلى `save`؟
- هل `EventBus` يعبر عملية أخرى؟

## الخلاصة

`CommandBus` و `QueryBus` مدخلان منتظران للكتابة والقراءة. `EventBus` يخبر داخل العملية بعد الحفظ. عالج الفشل في معالج الحدث صراحة لأنه لا يرجع على مسار HTTP.

## ماذا بعد؟

متى يفيد CQRS: اختلاف النموذج والآثار، لا حب الحافلات.

## المصادر الرسمية

- NestJS — CQRS: https://docs.nestjs.com/recipes/cqrs
