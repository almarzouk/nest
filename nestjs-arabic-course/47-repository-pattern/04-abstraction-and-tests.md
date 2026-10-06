# التجريد والاختبار

## الهدف من هذا الدرس

اختبار السياسة بمستودع في الذاكرة، واختبار ترجمة Prisma على قاعدة حقيقية بشكل أضيق، من غير تزوير العميل كله في كل ملف.

## المشكلة التي يحلها

تزوير `PrismaClient` بكل مولداته هش: الاختبار ينكسر عندما يتغير الاستدعاء لا عندما تتكسر القاعدة. تزوير `OrdersRepository` يثبت `cancel`. ترك SQL بلا أي اختبار تكامل يترك `snapshot` يكتب عمودًا خاطئًا. اقسم الاختبارات على الحد الذي رسمته.

## الفكرة ببساطة

وحدة السياسة: `useValue` على `ORDERS` بكائن يحفظ `Map`. لا `TestingModule` إن كان `new CancelOrder(repo)` يكفي. وحدة التنفيذ: اختبار أقل عددًا ضد Postgres، أو حاوية في CI، تتأكد أن `save` ثم `findById` يعيدان الحالة. لا تكرر جدول حالات الإلغاء في اختبار SQL. تلك الحالات على الكيان.

## التعريف التقني

```ts
import { Order } from './order.js';
import type { OrdersRepository } from './orders.repository.js';

export function inMemoryOrders(): OrdersRepository {
  const rows = new Map<string, Order>();
  return {
    async findById(id) {
      return rows.get(id) ?? null;
    },
    async save(order) {
      const snap = order.snapshot();
      rows.set(snap.id, new Order(snap.id, snap.status));
    },
  };
}
```

```ts
const orders = inMemoryOrders();
await orders.save(new Order('o1', 'open'));
const cancel = new CancelOrder(orders);
await cancel.execute('o1');
const stored = await orders.findById('o1');
```

بعد `execute` الحالة الملغاة في الخريطة. استدعاء ثانٍ يرمي. هذا اختبار ليسكوف للمزيف: `findById` يعيد `null` للغياب لا رميًا.

اختبار Nest عندما تحتاج الحارس أو المتحكم:

```ts
await Test.createTestingModule({
  controllers: [OrdersController],
  providers: [CancelOrder, { provide: ORDERS, useValue: inMemoryOrders() }],
}).compile();
```

لا تسجل `PrismaService` في هذا الاختبار.

اختبار التنفيذ علامة واحدة تكفي للمسار السعيد للحفظ، مع قاعدة معزولة. إن كان المشروع صغيرًا وما زال `prisma` داخل الخدمة بلا منفذ، زوّر الدوال التي تناديها الخدمة (`book.findUnique`) لا الكائن المولَّد كله، أو ارفع قاعدة اختبار. لا تبنِ منفذًا فقط ليصبح المزيف أسهل إن كان لديك اختبار واحد.

`useExisting` لا يبدّل التنفيذ في الاختبار. غطِّ الرمز `ORDERS`.

## Mental Model

```text
كيان + CancelOrder + Map     -> كثير، بلا Docker
PrismaOrdersRepository       -> قليل، مع قاعدة
Controller + inMemory         -> عقد HTTP الحرج
```

الهرم أثقل عند السياسة وأخف عند الترجمة.

## أبسط مثال

الدالتان أعلاه. تأكيد: `stored?.snapshot().status === 'cancelled'`. لا تتصل بـ `localhost`. إن فشل التأكيد فالعيب في `cancel` أو في المستودع الذاكرة، وكلاهما بلا شبكة.

## شرح المثال

`save` في الذاكرة يخزن نسخة من اللقطة حتى لا يشارك الاختبار مرجعًا يغيّره لاحقًا بصمت. هذا يحاكي أن الجلب التالي يعيد حالة محفوظة لا المؤشر نفسه إن كان ذلك وعد التنفيذ الحقيقي. طابق الوعد الذي كتبته في العقد.

## ماذا يحدث داخليًا؟

1. `Map` تعيش داخل إغلاق الدالة لكل اختبار.
2. `CancelOrder` لا يعرف نوع التخزين.
3. `execute` يجلب ويعدّل ويحفظ.
4. الجلب الثاني يقرأ ما كُتب.
5. اختبار Prisma يشغّل SQL حقيقيًا ويتأكد من العمود.
6. تغطية `PrismaService` بـ `useValue` جزئي تصلح عندما لا يوجد منفذ بعد، وتكسر عندما تستدعي الخدمة دالة لم تكتبها على المزيف.

## مثال عملي

مسار `POST /orders/:id/cancel` مع `supertest` و `inMemoryOrders` يثبت `404` عندما `findById` يعيد `null` و `200` عند الإلغاء. اختبار واحد على Postgres يثبت أن `status` في الجدول يصير `cancelled`. لا تشغّل العشرة سيناريوهات التجارية على Postgres. إن كتب التنفيذ استثناء HTTP فانقله قبل أن تعتمد على هرم الاختبار هذا.

## أخطاء شائعة

- مزيف Prisma ضخم يتتبع شكل المولّد.
- اختبار سياسة يحتاج Docker.
- `Map` واحدة على مستوى الملف تسرّب حالة بين الاختبارات.
- إعادة كيان Prisma من مزيف الدومين.
- حذف اختبار القاعدة كليًا فيضيع خطأ العمود.

## متى نستخدمه؟

فور وجود `OrdersRepository`. السياسة على الذاكرة، والترجمة على قاعدة بعدد صغير.

## متى لا نستخدمه؟

لا تزور مستودعًا لم تبنِه. زوّر حد الخدمة الحالي أو اختبر HTTP. ولا تغطِّ `ORDERS` في اختبار أردت منه تنفيذ Prisma.

## مقارنة

| الاختبار | البديل | يثبت |
| --- | --- | --- |
| `new CancelOrder(memory)` | منفذ | قواعد الجذر |
| `TestingModule` بلا Prisma | المنفذ أيضًا | عقد المتحكم |
| Prisma ضد Postgres | لا بديل | الترجمة والقيود |

## تمارين

### تمرين سهل

لماذا `Map` داخل الدالة لا خارجها؟

<details>
<summary>الحل</summary>

حتى يأخذ كل اختبار خريطة جديدة ولا يرى صفوف اختبار سابق.

</details>

### تمرين متوسط

ماذا لا تعيد كتابته في اختبار Postgres؟

<details>
<summary>الحل</summary>

جدول قواعد الإلغاء. تلك على `Order` مع المستودع الذاكرة. اختبار القاعدة يثبت الحفظ والقراءة.

</details>

### تمرين متقدم

الخدمة ما زالت تحقن `PrismaService` مباشرة. كيف تختبر فرع «غير موجود» قبل أن تستخرج منفذًا؟

<details>
<summary>الحل</summary>

غطِّ الدالة المستخدمة فقط، مثل `{ book: { findUnique: async () => null } }` على رمز `PrismaService`، أو استخرج المنفذ عندما تتعدد الدوال ويهش التزوير.

</details>

## أسئلة مراجعة

- أي رمز يُغطى في اختبار السياسة؟
- ماذا يبقى على قاعدة حقيقية؟
- متى التزوير الجزئي لـ Prisma مؤقت مشروع؟

## الخلاصة

زوّر مستودع الدومين في اختبار السياسة، واترك لـ Prisma اختبار ترجمة قليلًا على قاعدة حقيقية. لا تزور العميل المولَّد كله ما دام الحد يمكن أن يكون مجموعة الجذر.

## ماذا بعد؟

أمثلة واجهة وتنفيذ ذاكرة وتنفيذ Prisma.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
