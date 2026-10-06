# Domain Repository

## الهدف من هذا الدرس

وضع واجهة المجموعة بجانب الدومين وجعل عميل Prisma تنفيذًا خلف الرمز، يركّب الجذر ويحفظه من غير أن يستورد السياسةُ العميل.

## المشكلة التي يحلها

`Repository<Entity>` أو `prisma.order.findUnique` يعيد شكل الأداة. قواعد `cancel` إن قرأت هذا الشكل تعلقت بالأعمدة. مستودع الدومين يعيد `Order` الذي تملكه، و `PrismaOrdersRepository` يترجم صفوفًا إلى هذا الكائن ثم يعيد كتابتها. السياسة تبقى على `findById` و `save`.

## الفكرة ببساطة

الواجهة والكيان في مجلد السياق. الصنف الذي يحقن `PrismaService` في مجلد البنية لنفس الوحدة أو لوحدة بنية يستوردها السياق عبر الربط فقط. الربط `{ provide: ORDERS, useClass: PrismaOrdersRepository }`. يمكن أن يبقى `PrismaService` Singleton كما بنيته في دورة الحياة. المستودع الدومين لا يعني عميلًا لكل طلب.

## التعريف التقني

```ts
export class Order {
  constructor(
    readonly id: string,
    private status: 'open' | 'cancelled',
  ) {}

  cancel(): void {
    if (this.status !== 'open') {
      throw new Error('لا يُلغى إلا المفتوح');
    }
    this.status = 'cancelled';
  }

  snapshot(): { id: string; status: 'open' | 'cancelled' } {
    return { id: this.id, status: this.status };
  }
}
```

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service.js';
import { Order } from './order.js';
import type { OrdersRepository } from './orders.repository.js';

@Injectable()
export class PrismaOrdersRepository implements OrdersRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({ where: { id } });
    if (row === null) {
      return null;
    }
    return new Order(row.id, row.status as 'open' | 'cancelled');
  }

  async save(order: Order): Promise<void> {
    const data = order.snapshot();
    await this.prisma.order.update({ where: { id: data.id }, data: { status: data.status } });
  }
}
```

`snapshot` يفضّل على جعل كل الحقول عامة إن كنت تحمي التغيير عبر `cancel`. التنفيذ يقرأ اللقطة لا أن يعيد كتابة الحالة متجاوزًا الدالة.

الواجهة:

```ts
export interface OrdersRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

export const ORDERS = Symbol('ORDERS');
```

وحدة السياق تسجل المحول والرمز وتصدّر خدمة التطبيق لا `PrismaOrdersRepository` إلا لاختبارات التكامل داخل الوحدة.

معاملة السطور: `save` يستعمل `prisma.$transaction` عندما يكتب أكثر من جدول. السياسة لا تفتح المعاملة nor تعرف أسماء الجداول.

## Mental Model

```text
PlaceOrder -> ORDERS -> PrismaOrdersRepository -> PrismaService
                 ^
              Order.cancel
```

العميل ورقة تحت التنفيذ. الكيان فوق الترجمة.

## أبسط مثال

```ts
@Injectable()
export class CancelOrder {
  constructor(@Inject(ORDERS) private readonly orders: OrdersRepository) {}

  async execute(id: string): Promise<void> {
    const order = await this.orders.findById(id);
    if (order === null) {
      throw new Error('الطلب غير موجود');
    }
    order.cancel();
    await this.orders.save(order);
  }
}
```

اختبار يمرر مستودعًا في الذاكرة يحفظ `Map`. `cancel` مرتين على المفتوح ثم الملغى يرمي من الكيان من غير Prisma.

## شرح المثال

`execute` لا يذكر `status` في عمود. تغيير اسم العمود يمس `PrismaOrdersRepository` و `snapshot` إن تغيّر شكل اللقطة المتفق مع الجدول. قاعدة «لا يُلغى إلا المفتوح» تبقى في `Order`. هذا تقسيم الدرسين: النمط والعكس.

## ماذا يحدث داخليًا؟

1. `findUnique` يجلب صفًا أو `null`.
2. التنفيذ يبني `Order` أو يعيد `null` بلا استثناء HTTP.
3. `cancel` يغيّر الحالة الداخلية.
4. `update` يكتب اللقطة.
5. فشل القيد في Prisma عطل بنية، لا «طلب غير موجود».
6. `PrismaService` متصل منذ `onModuleInit` على Singleton. المستودع لا ينادي `$connect` لكل `save`.

## مثال عملي

سطور الطلب داخل `save` بـ `$transaction`: احذف السطور القديمة أو حدّثها حسب لقطة الجذر، ثم حدّث الرأس. السياسة `addLine` ثم `save` مرة. لا تصدّر دالة `addLineRow` على المستودع. تقرير الفواتير يقرأ بـ `prisma` في `InvoiceQuery` من غير المرور على `Order`. القراءة للعرض ليست مجموعة الجذر.

## أخطاء شائعة

- الواجهة في ملف يستورد `PrismaClient`.
- إرجاع الصف من `findById`.
- `$connect` داخل `save`.
- تعليم المستودع `REQUEST` لأن الصفحة تحتاج المستخدم. مرر المعرف وسيطًا.
- معاملة في السياسة فيها استدعاءات Prisma مباشرة فينهار العزل.

## متى نستخدمه؟

عندما تعيش قواعد على الجذر وتريد مزيف `Map` في الاختبار، وPrisma يبقى الترجمة.

## متى لا نستخدمه؟

قراءة DTO بلا كيان. `prisma` في خدمة الوحدة أصغر. لا تبنِ كيانًا يعيد تعيين الحقول فقط ليبرر المستودع.

## مقارنة

| الطبقة | النوع الذي تعرفه |
| --- | --- |
| `CancelOrder` | `Order` و `OrdersRepository` |
| `PrismaOrdersRepository` | `Order` و Prisma |
| `PrismaService` | الجداول |
| `InvoiceQuery` | DTO القراءة، بلا جذر |

## تمارين

### تمرين سهل

لماذا `findById` يبني `new Order`؟

<details>
<summary>الحل</summary>

حتى ترى السياسة كيانها لا صف Prisma. الترجمة مكانها التنفيذ.

</details>

### تمرين متوسط

أين تُكتب معاملة الرأس والسطور؟

<details>
<summary>الحل</summary>

داخل `save` في التنفيذ، بـ `prisma.$transaction`. السياسة تنادي `save` مرة.

</details>

### تمرين متقدم

كيف تختبر «الإلغاء الثاني يرمي» بلا قاعدة؟

<details>
<summary>الحل</summary>

مستودع ذاكرة خلف `ORDERS` يعيد `Order` مفتوحًا ثم يحفظ الملغى. النداء الثاني على نفس الكيان يرمي من `cancel` قبل أي SQL.

</details>

## أسئلة مراجعة

- من يستورد واجهة المستودع؟
- ماذا يفعل `snapshot`؟
- لماذا العميل يبقى Singleton؟

## الخلاصة

مستودع الدومين واجهة مجموعة تملكها السياسة، وتنفيذ Prisma يترجم الجذر. العميل المشترك يبقى خلف التنفيذ، والقراءات العرضية تبقى استعلامات بجانب النمط.

## ماذا بعد؟

التجريد والاختبار: ماذا تزوّر، وماذا تترك لقاعدة حقيقية.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
