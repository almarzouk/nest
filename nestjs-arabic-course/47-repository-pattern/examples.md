# أمثلة عملية

## مجموعة للجذر

```ts
export interface OrdersRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

`cancel` على الكيان ثم `save`. لا `saveLine` من المتحكم.

## مساعد ORM يبقى في الحافة

```ts
constructor(@InjectRepository(BookEntity) private readonly books: Repository<BookEntity>) {}
```

الاسم `BooksStore` صادق. صفحة JSON تعيد DTO لا `BookEntity`.

## Prisma خلف المنفذ

```ts
async findById(id: string): Promise<Order | null> {
  const row = await this.prisma.order.findUnique({ where: { id } });
  if (row === null) return null;
  return new Order(row.id, row.status as 'open' | 'cancelled');
}
```

`PrismaService` Singleton. `save` يستعمل `$transaction` إن كتب أكثر من جدول.

## ذاكرة للاختبار

```ts
{ provide: ORDERS, useValue: inMemoryOrders() }
```

قواعد الإلغاء على هذا المزيف. اختبار واحد على Postgres يتحقق من العمود بعد `save`.

## ماذا تتأكد منه

1. واجهة الدومين لا تستورد `@prisma/client`.
2. الغياب `null` في Prisma وفي الذاكرة.
3. تقرير المبيعات ليس دالة على مستودع الجذر.
4. لا `Scope.REQUEST` على العميل لأجل المستخدم.
