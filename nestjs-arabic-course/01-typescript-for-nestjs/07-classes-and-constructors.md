# Classes و Constructor

## الهدف من هذا الدرس

قراءة صنف Nest كقالب كائنات: حقول، وconstructor يهيئها، ودوال تعمل على `this`.

## المشكلة التي يحلها

الدوال المتفرقة التي تشارك متغيرًا عامًا `let books = []` تخفي من يملك البيانات. الصنف يجمع الحالة والسلوك في قالب واحد، والإطار يصنع منه كائنًا.

## الفكرة ببساطة

`class` قالب. `new` يصنع كائنًا ويستدعي `constructor`. داخل الدوال، `this` هو ذلك الكائن.

في Nest لن تكتب `new UsersService()` في المتحكم. الحاوية تفعل ما يفعله `new` مع حقن الاعتماديات. افهم `new` أولًا حتى ترى ماذا أُخذ منك عمدًا.

## التعريف التقني

الصنف في JavaScript دالة بناء مع prototype. TypeScript يضيف الأنواع ومعدّلات الوصول. `constructor` دالة التهيئة، وتُستدعى مرة لكل `new`. لا تُرجع الكائن يدويًا في الاستخدام العادي؛ `new` يعيد الكائن.

## Mental Model

```text
class BooksService     القالب
new BooksService()     كائن واحد
this.books             حالة هذا الكائن
container.create       لاحقًا بدل new في المتحكم
```

## أبسط مثال

```ts
class BooksService {
  private readonly books: string[] = [];

  add(title: string): void {
    this.books.push(title);
  }

  count(): number {
    return this.books.length;
  }
}

const service = new BooksService();
service.add('Refactoring');
```

## شرح المثال

- `books` حقل على كل كائن، ومصفوفة جديدة لكل `new` لأن التهيئة على الحقل.
- `private` فحص TypeScript: من خارج الصنف لا نكتب `service.books`. وقت التشغيل الاسم ما زال موجودًا على الكائن ما لم تستخدم حقول `#` الخاصة في JavaScript.
- `readonly` يمنع استبدال المصفوفة، لا `push`.
- `add` و `count` دوال على prototype، و `this` داخلها هو الكائن الذي نُوديت عليه.
- `new BooksService()` يشغّل التهيئة. لا يوجد constructor مكتوب، فهناك واحد افتراضي فارغ.

constructor صريح:

```ts
class Clock {
  constructor(private readonly startedAt: Date) {}
}
```

هذا الاختصار درس 09. شكله المطول:

```ts
class Clock {
  private readonly startedAt: Date;

  constructor(startedAt: Date) {
    this.startedAt = startedAt;
  }
}
```

المعامل يُمرَّر إلى `new Clock(new Date())` ويُنسخ إلى الحقل.

## ماذا يحدث داخليًا؟

`new` ينشئ كائنًا، يربطه بـ prototype الصنف، ويستدعي constructor مع الوسائط. إن رمى constructor، لا يُسلَّم إليك كائن نصف مهيأ في الاستخدام العادي.

نسختان من `BooksService` تعنيان مصفوفتين. في Nest، التسجيل الافتراضي ينشئ نسخة واحدة للتطبيق كله. لذلك حالة على الحقل مثل هذه المصفوفة تصبح حالة مشتركة بين كل الطلبات. مقبولة لتجربة في الذاكرة، وخطرة إذا حسبتها "بيانات هذا المستخدم".

## مثال عملي

```ts
class Price {
  constructor(private readonly amount: number) {}

  withDiscount(rate: number): number {
    return this.amount * (1 - rate);
  }
}

const price = new Price(40);
price.withDiscount(0.25);
```

`amount` لا يتغير. الدالة تحسب ناتجًا. هذا نمط يناسب خدمات بلا حالة مشتركة قابلة للتغيير: المعطيات تدخل، والناتج يخرج، والمخزن الحقيقي قاعدة البيانات لا حقل على الخدمة.

## أخطاء شائعة

- نسيان `this.` وقراءة متغير حر بالاسم نفسه.
- وضع طلب HTTP كامل على الحقل في خدمة singleton. الطلب التالي يرى بقايا السابق.
- `new` للخدمة داخل كل دالة متحكم. تفقد المشاركة والاختبار. الحقن سيحل محله، لا تبنِ العادة.

## متى نستخدم صنفًا؟

لمتحكمي Nest وخدماتها ووحداتها. الإطار يطلب أصنافًا ليلصق عليها Decorators.

## متى لا نستخدمه؟

لدالة تحويل نقية بلا حالة. دالة عادية أوضح. Nest تسمح بمزوّدات غير صنفية لاحقًا، بعد أن تفهم الصنف.

## مقارنة

| | دالة + متغير عام | صنف |
| --- | --- | --- |
| من يملك الحالة | الملف | الكائن |
| من ينشئ في Nest | الاستيراد | الحاوية غالبًا |
| الاختبار | يستبدل العام | يبني الكائن بمعطيات |

## تمارين

### تمرين سهل

ماذا يطبع `count()` بعد `add` مرتين على الكائن نفسه؟

<details>
<summary>الحل</summary>

`2`. المصفوفة على الكائن تتراكم.

</details>

### تمرين متوسط

اكتب الصنف المطول المكافئ لـ `constructor(private readonly amount: number)`.

<details>
<summary>الحل</summary>

```ts
class Price {
  private readonly amount: number;
  constructor(amount: number) {
    this.amount = amount;
  }
}
```

</details>

### تمرين متقدم

لماذا نسختان من `BooksService` في المثال لا تريان كتب بعضهما، بينما نسخة Nest الافتراضية الواحدة يراها كل طلب؟

<details>
<summary>الحل</summary>

كل `new` يهيئ حقل `books` من جديد. الحاوية الافتراضية لا تنادي `new` لكل طلب، بل تحتفظ بكائن واحد. الحقل القابل للتغيير على ذلك الكائن مشترك. لذلك الحالة الخاصة بالطلب لا تُخزَّن على خدمة singleton.

</details>

## أسئلة مراجعة

- متى يشتغل constructor؟
- ماذا يشير إليه `this` داخل دالة الصنف؟
- لماذا نسخة الخدمة الافتراضية في Nest تجعل الحقل المشترك خطرًا لبيانات طلب واحد؟

## الخلاصة

الصنف قالب، والconstructor يهيئ الكائن، و `this` حالته. Nest تنشئ هذه الكائنات في الحاوية. افصل حالة الطلب عن حقول الخدمة المشتركة.

## ماذا بعد؟

`private` ظهر بسرعة. الدرس التالي يفرقه عن `public` و `protected`.

## المصادر الرسمية

- TypeScript — Classes: https://www.typescriptlang.org/docs/handbook/2/classes.html
- NestJS — Providers: https://docs.nestjs.com/providers
