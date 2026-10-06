# أمثلة عملية

## CRUD يبقى دالة

```ts
@Post()
create(@Body() dto: CreateBookDto) {
  return this.books.create(dto);
}
```

لا `CreateBookCommand` ما دام `GET` يعيد نفس الشكل.

## أمر وإلغاء

```ts
@CommandHandler(CancelOrderCommand)
export class CancelOrderHandler implements ICommandHandler<CancelOrderCommand> {
  async execute(command: CancelOrderCommand): Promise<void> {
    const order = await this.orders.findById(command.orderId);
    if (order === null) throw new Error('الطلب غير موجود');
    order.cancel();
    await this.orders.save(order);
  }
}
```

المتحكم: `commandBus.execute(new CancelOrderCommand(id))`. المعالج في `providers` و `CqrsModule.forRoot()` في الجذر.

## حدث بعد الحفظ

```ts
await this.orders.save(order);
this.eventBus.publish(new OrderCancelled(order.snapshot().id));
```

`@EventsHandler(OrderCancelled)` يحدّث `order_view`. `GetOrderQuery` يقرأ العرض. فشل المعالج لا يعتمد عليه جسم HTTP.

## ماذا تتأكد منه

1. لا معالجات في وحدة الكتب إن كانت CRUD.
2. الاستعلام لا يستدعي `save`.
3. `userId` حقل في الأمر لا `Scope.REQUEST` على المعالج.
4. حدث بلا مشترك يُحذف.
