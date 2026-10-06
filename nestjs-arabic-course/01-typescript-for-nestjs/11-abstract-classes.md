# Abstract Classes

## الهدف من هذا الدرس

قراءة `abstract` كعقد على أصناف وسيطة: دوال يجب أن يملأها الابن، وأساس لا يُنشأ بـ `new` مباشرة.

## المشكلة التي يحلها

واجهة Interface تفرض دوالًا وتُمحى. أحيانًا تريد أساسًا فيه كود مشترك حي، وفيه فجوة يملأها كل ابن بكود مختلف. إن سمحنا بـ `new` على الأساس، الفجوة تُستدعى فارغة.

## الفكرة ببساطة

`abstract class` صنف ناقص. `abstract` على دالة تعني: الابن يكتب الجسم. `new` يكون على الابن المكتمل.

## التعريف التقني

الصنف المجرد يعيش وقت التشغيل كدالة بناء مثل أي صنف، لكن TypeScript يمنع `new` عليه مباشرة. الدالة المجردة بلا جسم في الأب. الابن غير المجرد يجب أن ينفذ كل الدوال المجردة وإلا يبقى مجردًا.

## Mental Model

```text
abstract Repository
   abstract save()
   protected log() { ... كود حي }

SqlRepository extends Repository
   save() { ... SQL }

new SqlRepository()      مسموح
new Repository()         مرفوض في الفحص
```

## أبسط مثال

```ts
abstract class Repository<T> {
  abstract save(entity: T): Promise<T>;

  protected label(): string {
    return this.constructor.name;
  }
}

class MemoryBooks extends Repository<{ title: string }> {
  async save(entity: { title: string }): Promise<{ title: string }> {
    return entity;
  }
}
```

`Repository<T>` يستخدم Generic، والدرس التالي يفك الرمز `<T>`. هنا يكفي أن `T` عنصر نائب للنوع.

## شرح المثال

- لا جسم لـ `save` في الأب. لا يوجد سلوك افتراضي صادق.
- `label` كود مشترك حي يستخدم اسم الصنف الابن وقت التشغيل.
- `MemoryBooks` ينفّذ `save` فيصبح قابلًا للإنشاء.
- `protected` تبقي `label` للأبناء.

## ماذا يحدث داخليًا؟

JavaScript الناتج فيه الصنفان. المنع على `new Repository` فحص مترجم. الاختبار والحاوية يتعاملان مع الابن أو مع رمز حقن يشير إلى عقد. في Nest، حقن صنف مجرد مباشرة لا ينشئ الابن الصحيح من غير تسجيل يربط الرمز بالتنفيذ (`useClass`). هذا موضوع Custom Providers. الآن: المجرد عقد + كود مشترك، والتنفيذ صنف ملموس.

فصل Repository لاحقًا قد يستخدم هذا الشكل. لا تبدأ به في أول خدمة كتب. صنف ملموس واحد يكفي حتى تظهر حاجة لاستبدال المخزن في الاختبار بهذا الأسلوب.

## مثال عملي

سياسة سعر يشاركها أكثر من نوع، والحساب يختلف:

```ts
abstract class Pricing {
  abstract unit(price: number): number;

  total(price: number, quantity: number): number {
    return this.unit(price) * quantity;
  }
}

class FullPrice extends Pricing {
  unit(price: number): number {
    return price;
  }
}
```

`total` مكتوب مرة. `unit` قرار الابن. بديل بالتركيب: دالة `unit` تُمرَّر في constructor. إذا كان الاختلاف دالة واحدة، التركيب أو حتى معامل دالة أبسط من صنف مجرد. المجرد يستحق عندما تتعدد الدوال المشتركة والكود الحي معًا.

## أخطاء شائعة

- صنف مجرد لكل خدمة من اليوم الأول.
- نسيان تنفيذ دالة مجردة في الابن.
- حقن الرمز المجرد في Nest من غير ربطه بصنف ملموس، ثم استغراب فشل الإقلاع لاحقًا.

## متى نستخدمه؟

عند أساس فيه تنفيذ مشترك وفجوات حقيقية، أو عند قراءة مكتبة تستخدم هذا الأسلوب.

## متى لا نستخدمه؟

لعقد بلا كود حي. Interface أو type أخف. وللاختلاف البسيط الذي يغطيه حقن مزوّد آخر.

## مقارنة

| | Interface | Abstract class |
| --- | --- | --- |
| كود حي مشترك | لا | نعم |
| وقت التشغيل | تُمحى | صنف |
| `new` | لا | على الابن فقط |

## تمارين

### تمرين سهل

لماذا لا نكتب `new Pricing()` في المثال؟

<details>
<summary>الحل</summary>

لأن `unit` بلا جسم. الصنف ناقص، والمترجم يمنع إنشاءه.

</details>

### تمرين متوسط

أضف ابنًا `HalfPrice` يجعل `unit` يعيد نصف السعر.

<details>
<summary>الحل</summary>

```ts
class HalfPrice extends Pricing {
  unit(price: number): number {
    return price / 2;
  }
}
```

`total(40, 2)` على كائن منه يعيد `40`.

</details>

### تمرين متقدم

حوّل المثال إلى تركيب من غير `abstract`: صنف يأخذ دالة `unit` في constructor. متى يصبح هذا أوضح؟

<details>
<summary>الحل</summary>

```ts
class Pricing {
  constructor(private readonly unit: (price: number) => number) {}
  total(price: number, quantity: number): number {
    return this.unit(price) * quantity;
  }
}
```

أوضح عندما الاختلاف دالة واحدة ويمكن تمريرها في الاختبار من غير أصناف أبناء. المجرد أوضح عندما الأبناء أنواع مجال بأسماء ثابتة وعدة دوال.

</details>

## أسئلة مراجعة

- ماذا يلزم الابن غير المجرد؟
- ما الفرق بين Interface وصنف مجرد فيه دالة فيها جسم؟
- لماذا لا نحقن المجرد في Nest من غير ربط تنفيذ؟

## الخلاصة

الصنف المجرد أساس حي ناقص. الابن يملأ الفجوات. استخدمه بقلة، وفكّر أولًا في صنف ملموس أو حقن.

## ماذا بعد؟

`Repository<T>` ظهر كرمز. Generics تشرح العنصر النائب.

## المصادر الرسمية

- TypeScript — Abstract classes: https://www.typescriptlang.org/docs/handbook/2/classes.html#abstract-classes
- NestJS — Custom providers (ربط تنفيذ بعقد، لاحقًا): https://docs.nestjs.com/fundamentals/custom-providers
