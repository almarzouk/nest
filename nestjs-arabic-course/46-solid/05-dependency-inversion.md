# Dependency Inversion

## الهدف من هذا الدرس

قلب اتجاه الاعتماد في Nest: سياسة الطلبات تعتمد على رمز منفذ تملكه، ومحول Prisma هو الذي يعتمد على ذلك الرمز وينفذه.

## المشكلة التي يحلها

`OrdersService` تستورد `PrismaService` فتصبح السياسة تابعة لتفصيل أدنى. كل تغيير عميل أو اختبار يحتاج القاعدة أو مزيفًا يشبه العميل كله. مبدأ عكس الاعتماد: الوحدات العالية لا تعتمد على وحدات منخفضة. كلاهما يعتمد على تجريد. التفاصيل تعتمد على التجريد أيضًا. الحاوية في Nest هي مكان الوصل: `provide` التجريد و `useClass` التفصيل.

## الفكرة ببساطة

ضع `OrdersPort` و `Symbol` بجانب `PlaceOrder`. المحول في مجلد بنية يستورد المنفذ. الوحدة تربط `{ provide: ORDERS, useClass: PrismaOrdersAdapter }`. `PlaceOrder` لا يظهر في ملفها مسار Prisma. هذا ما مهّد له Clean، والمبدأ هنا اسم الضغط: اعكس السهم.

## التعريف التقني

اتجاه خاطئ للتطبيق الذي قرر العزل:

```ts
import { PrismaService } from '../prisma/prisma.service.js';
```

داخل `PlaceOrder`.

اتجاه معكوس:

```ts
import { Inject, Injectable } from '@nestjs/common';
import { ORDERS, type OrdersPort } from './orders.port.js';

@Injectable()
export class PlaceOrder {
  constructor(@Inject(ORDERS) private readonly orders: OrdersPort) {}

  execute(totalMinor: number) {
    if (totalMinor < 0) {
      throw new Error('المجموع سالب');
    }
    return this.orders.save({ totalMinor });
  }
}
```

`PrismaOrdersAdapter implements OrdersPort` ويستورد Prisma. السهم من الملف الأدنى إلى ملف المنفذ في الأعلى.

الحقن بالـ `constructor` هو عكس التحكم الذي بنيت عليه الدورة: الصنف لا يعمل `new` لتفصيله. المبدأ يزيد أن نوع ذلك الاعتماد تجريد يملكه الأعلى.

في المشروع الصغير الذي أبقى Prisma داخل الخدمة عمدًا، الاعتماد ما زال على تفصيل، وهو مقبول ما دمت وثّقت التأجيل. عندما تلمس `import { PrismaClient }` من ملف سياسة تريد اختباره يوميًا، اعكس ذلك الملف فقط.

`forwardRef` بين سياسة وتفصيل علامة أن السهم ما زال متشابكًا. بعد العكس، السياسة لا تستورد المحول فلا دورة.

## Mental Model

```text
PlaceOrder -> OrdersPort
                 ^
PrismaOrdersAdapter
الوحدة: ORDERS = PrismaOrdersAdapter
```

الأعلى يسمي الحاجة. الأدنى يلبّيها. الحاوية تعرّف من يلبّي اليوم.

## أبسط مثال

اختبار:

```ts
const saved: number[] = [];
const orders: OrdersPort = {
  save: async (input) => {
    saved.push(input.totalMinor);
  },
};
const place = new PlaceOrder(orders);
await place.execute(100);
```

لا حاوية ولا قاعدة. المحول الحقيقي يبقى لملف البنية واختبار التكامل.

## شرح المثال

`execute( -1 )` يرمي قبل `save`. هذا اختبار سياسة لا اختبار SQL. العكس هو الذي جعل `new PlaceOrder(orders)` ممكنًا بمعامل بسيط. خدمة تبني `new PrismaClient()` داخل الدالة لا تُعطى هذا المعامل.

## ماذا يحدث داخليًا؟

1. سياسة الإقلاع تسجل الربط في الوحدة.
2. بناء `PlaceOrder` يطلب `ORDERS`.
3. الحاوية تبني المحول وتحل Prisma داخله.
4. وقت النداء تعبر الدالة المنفذ إلى المحول.
5. استبدال الربط في `TestingModule` يضع المزيف.
6. المحول الذي يستدعي `PlaceOrder` يعيد السهم ويصنع دورة. الأثر الجانبي بعد الحفظ يبقى خارج السياسة أو في منفذ خروج آخر.

## مثال عملي

وحدة `OrdersModule`:

```ts
@Module({
  providers: [
    PlaceOrder,
    PrismaOrdersAdapter,
    { provide: ORDERS, useExisting: PrismaOrdersAdapter },
  ],
  exports: [PlaceOrder],
})
export class OrdersModule {}
```

المتحكم في الوحدة أو في وحدة مدخل يستورد `OrdersModule` ويحقن `PlaceOrder`. لا يحقن المحول. تصدير `PlaceOrder` سطح الدخول. إبقاء `ORDERS` غير مصدَّر يمنع الجار من استبدال المخزن. إن احتاج اختبار خارجي الاستبدال، فالاختبار يعيد الربط في `TestingModule` ولا يعتمد على تصدير الرمز للإنتاج.

## أخطاء شائعة

- تجريد في ملف يستورد Prisma.
- سياسة تعمل `new PrismaOrdersAdapter()`.
- تصدير المحول كسطح الوحدة.
- عكس كل استعلام قراءة قبل وجود اختبار أو محول ثانٍ.
- واجهة تعيد أنواع العميل فيبقى السهم المنطقي نازلًا.

## متى نستخدمه؟

عندما يجب أن تُختبر السياسة أو تُقاد من غير التفصيل، وعلى حدود السياق التي استقرت.

## متى لا نستخدمه؟

لا تعكس استدعاء `ConfigService` الإعدادي في كل مكان. الإعداد تفصيل تطبيق مقبول في الحافة. ولا تعكس `Logger` إن كان عقد Nest كافيًا للفريق.

## مقارنة

| الاتجاه | من يستورد من | الاختبار |
| --- | --- | --- |
| خدمة → Prisma | الأعلى يسمي التفصيل | غالبًا قاعدة أو مزيف ضخم |
| خدمة → منفذ ← محول | الأدنى يطبّق الأعلى | مزيف صغير على الرمز |

## تمارين

### تمرين سهل

من يملك ملف `OrdersPort`؟

<details>
<summary>الحل</summary>

جانب السياسة، لا جانب Prisma. المحول يستورد ذلك الملف.

</details>

### تمرين متوسط

لماذا `useExisting` في مثال الوحدة بدل `useClass` ثانٍ؟

<details>
<summary>الحل</summary>

لأن `PrismaOrdersAdapter` مسجّل أصلًا. الربط باللقب يبقي نسخة واحدة خلف `ORDERS`.

</details>

### تمرين متقدم

اختبار إنتاج يستورد الوحدة ويريد مزيفًا. أين تغيّر الربط من غير تصدير `ORDERS` للجار؟

<details>
<summary>الحل</summary>

في `Test.createTestingModule` أعد تعريف `{ provide: ORDERS, useValue: fake }` داخل حاوية الاختبار. تصدير الإنتاج يبقى `PlaceOrder` فقط.

</details>

## أسئلة مراجعة

- ماذا يعتمد عليه الأعلى؟
- ماذا يفعل `new` داخل السياسة؟
- متى إبقاؤك على Prisma داخل الخدمة قرار تأجيل لا سهو؟

## الخلاصة

اعكس الاعتماد عندما يضر السهم النازل: السياسة تطلب رمزًا تملكه، والمحول الأدنى ينفذه، والوحدة تربطهما. الحقن هو الآلية، ومكان الواجهة هو القرار.

## ماذا بعد؟

أمثلة SOLID على وحدة طلبات واحدة.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
- NestJS — Modules: https://docs.nestjs.com/modules
