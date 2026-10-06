# Open/Closed

## الهدف من هذا الدرس

إضافة سلوك جديد بتسجيل مزود يطبّق عقدًا، مع إبقاء المنسق مغلقًا أمام تعديل الفروع كلما ظهرت قناة دفع أو نوع شحن.

## المشكلة التي يحلها

`if (type === 'card') ... else if (type === 'cash')` داخل `CheckoutService` يتضخم مع كل مزود. كل إضافة تفتح الملف الأكثر حساسية. مبدأ الفتح والإغلاق: الكود مفتوح للتوسيع ومغلق للتعديل. في Nest التوسيع مزود جديد في `providers`، والتعديل الذي يبقى هو سطر التسجيل لا سلسلة الشروط.

## الفكرة ببساطة

عرّف عقد `Payment` بدالة `pay`. كل قناة صنف `@Injectable` يطبّق العقد. المنسق يستقبل القائمة أو يختار من خريطة حقنتها. إضافة المحفظة صنف جديد ورمز في الوحدة. `pay` في المنسق لا يكتسب فرعًا.

## التعريف التقني

```ts
export interface Payment {
  readonly code: string;
  pay(totalMinor: number): Promise<void>;
}

export const PAYMENTS = Symbol('PAYMENTS');
```

```ts
import { Injectable } from '@nestjs/common';
import type { Payment } from './payment.js';

@Injectable()
export class CardPayment implements Payment {
  readonly code = 'card';

  async pay(totalMinor: number): Promise<void> {
    void totalMinor;
  }
}
```

التسجيل بمصفوفة عبر مصنع أو بحقن متعدد إن استعملت نمطًا يدويًا:

```ts
{
  provide: PAYMENTS,
  useFactory: (card: CardPayment, cash: CashPayment) => [card, cash],
  inject: [CardPayment, CashPayment],
}
```

```ts
@Injectable()
export class CheckoutService {
  constructor(@Inject(PAYMENTS) private readonly payments: Payment[]) {}

  async charge(code: string, totalMinor: number): Promise<void> {
    const payment = this.payments.find((item) => item.code === code);
    if (!payment) {
      throw new Error(`قناة غير معروفة: ${code}`);
    }
    await payment.pay(totalMinor);
  }
}
```

إضافة `WalletPayment`: صنف، وعنصر في `inject` والمصفوفة. جسم `charge` ثابت. هذا الإغلاق العملي. تعديل العقد `pay` نفسه ما زال تغييرًا على الجميع، فاجعل العقد صغيرًا.

لا تحتاج `DiscoveryService` لقناتين. عندما تصبح القنوات إضافات بلا لمس الوحدة، المسح بالمزخرف يوسع التسجيل. حتى ذلك الحين القائمة صريحة وأوضح.

## Mental Model

```text
عقد ثابت
  CardPayment   \
  CashPayment    -> CheckoutService.charge يبحث بالرمز
  WalletPayment /
```

الملف الذي يكبر هو مجلد القنوات لا المنسق.

## أبسط مثال

المقاطع أعلاه. اختبار يمرر `useValue` بمصفوفة فيها مزيف `code: 'card'`. لا فرع بطاقة داخل الاختبار ولا داخل `charge`.

## شرح المثال

`find` اختيار من قائمة مفتوحة. الفشل عند رمز مجهول صريح. لا `default` صامت يخصم من بطاقة. تسجيل القناة في الوحدة هو نقطة التعديل الوحيدة المقصودة، وهي بيانات تكوين لا سياسة.

## ماذا يحدث داخليًا؟

1. الحاوية تبني كل تنفيذ.
2. المصنع يجمعها في مصفوفة خلف `PAYMENTS`.
3. `charge` يبحث ولا يعرف الأصناف بالاسم.
4. تنفيذ جديد يُبنى مع الإقلاع إن أُضيف إلى `inject`.
5. نسيان التسجيل يعني رمزًا مجهولًا عند النداء، لا مسارًا قديمًا صامتًا.
6. العقد الذي يعيد أنواع Stripe يكسر الإغلاق لأن المنسق سيتعرف على التفاصيل.

## مثال عملي

شحن: `ShippingQuote` بعقد `quote(weight): Money`. `StandardShipping` و `ExpressShipping`. `CheckoutService` يطلب العرض بالرمز الذي اختاره الزبون. تغيير سعر السريع يمس صنفًا واحدًا. إضافة نقطة استلام صنف ثالث. لا تفتح `CheckoutService` إلا إذا تغيّر تسلسل «احسب ثم ادفع ثم احفظ».

## أخطاء شائعة

- عقد فيه دالة لكل قناة.
- `switch` يبقى وينادي الأصناف، فالتعديل المزدوج.
- اكتشاف أصناف من القرص بلا تسجيل ظاهر فيستغرب الفريق من يعمل في الإنتاج.
- وراثة عميقة بدل تركيب مزودات.
- جعل المنسق `global` حتى «يرى» كل القنوات. الاستيراد أوضح.

## متى نستخدمه؟

عائلة سلوك تتسع: دفع، شحن، تصدير ملف، إشعار.

## متى لا نستخدمه؟

فرعان ثابتان لن يتغيرا. `if` أقصر من أربعة ملفات. لا تجرد قبل النسخة الثالثة إن كان الفريق ما زال يكتشف القاعدة.

## مقارنة

| الأسلوب | إضافة قناة |
| --- | --- |
| `switch` في الخدمة | تعديل الخدمة |
| قائمة `Payment` | صنف وتسجيل |
| وراثة قاعدة ضخمة | تعديل الأب غالبًا |

## تمارين

### تمرين سهل

ماذا يبقى ثابتًا عند إضافة `WalletPayment`؟

<details>
<summary>الحل</summary>

جسم `charge` الذي يبحث بـ `code` وينادي `pay`. يتغير الصنف الجديد وسطر التسجيل.

</details>

### تمرين متوسط

لماذا عقد `pay` الذي يعيد كائن Stripe يضعف الإغلاق؟

<details>
<summary>الحل</summary>

لأن المنسق سيكتب فروعًا على حقول Stripe، فكل قناة جديدة قد تعدّل المنسق من جديد.

</details>

### تمرين متقدم

متى تستبدل المصنع الصريح بـ `DiscoveryService`؟

<details>
<summary>الحل</summary>

عندما تُضاف القنوات من وحدات لا يجوز أن تعدّل وحدة الدفع عند كل إضافة، مع مزخرف ومسح عند الإقلاع. لقناتين أو ثلاث القائمة الصريحة أصدق.

</details>

## أسئلة مراجعة

- ما الذي يُغلق وما الذي يُفتح؟
- أين سطر التعديل المقبول؟
- متى `if` أفضل من عقد؟

## الخلاصة

وسّع السلوك بصنف يطبّق عقدًا صغيرًا وسجّله في الوحدة. أبقِ المنسق على الاختيار والنداء، وعدّل العقد فقط عندما تتغير السياسة المشتركة.

## ماذا بعد؟

Liskov Substitution: التنفيذ البديل يجب أن يحترم عقد المستدعي.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
