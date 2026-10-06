# أمثلة عملية

## مسؤولية HTTP مقابل السياسة

```ts
@Post()
create(@CurrentUserId() userId: string, @Body() dto: CreateOrderDto) {
  return this.orders.place(userId, dto);
}
```

الضريبة والبريد ليسا في المتحكم. البريد صنف `ReceiptMailer` إن استقل سبب تغييره.

## قنوات بلا switch

```ts
const payment = this.payments.find((item) => item.code === code);
await payment.pay(totalMinor);
```

`WalletPayment` يُضاف في `inject` لا داخل `charge`.

## استبدال سليم

`findTitle` يعيد `null` من Prisma ومن المزيف. الخدمة تترجم `null` عند الحافة. لا `NotFoundException` من المحول.

## رمزان ونسخة واحدة

```ts
{ provide: BOOKS_READ, useExisting: PrismaBooks },
{ provide: BOOKS_WRITE, useExisting: PrismaBooks },
```

التقرير يحقن `BOOKS_READ` فقط.

## عكس السهم

`PlaceOrder` يحقن `ORDERS`. `PrismaOrdersAdapter` يستورد المنفذ. الوحدة تصدّر `PlaceOrder` لا المحول.

## ماذا تتأكد منه

1. ملف السياسة بلا مسار `prisma`.
2. المزيف يمرر فرع الغياب لا النجاح فقط.
3. عقد الدفع لا يعيد أنواع مزود خارجي.
4. التصدير أضيق من الصنف الذي ينفذ كل الدوال.
