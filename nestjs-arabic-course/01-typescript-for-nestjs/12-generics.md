# Generics

## الهدف من هذا الدرس

قراءة الرمز `<T>` في أنواع Nest مثل `Promise<Book>` ومزوّدات لاحقة، وكتابة دالة لا تكرر نفسها لكل نوع.

## المشكلة التي يحلها

دالة `firstBook` و `firstUser` جسماهما واحد: أعد العنصر الأول أو `null`. نسخ الدالة لكل نوع يفرقهما يوم يصلح أحد النسخ فقط.

## الفكرة ببساطة

Generic معامل نوع. تكتبه بين `< >` وتستخدمه مكان نوع ثابت. المستدعي يحدد النوع، أو يستنتجه المترجم.

## التعريف التقني

Generics تعدد أشكال على الأنواع مع بقاء العلاقات: إذا دخل `T[]` خرج `T | null`. القيد `T extends { id: string }` يحد ما يمكن أن يكون `T`. وقت التشغيل لا يوجد `T`. التوثيق يقول أيضًا إن Metadata لا تُصدر للـ generics بشكل يفيد `ValidationPipe`. لا تجعل DTO عامًا وتتوقع فحص حقوله.

## Mental Model

```text
function first<T>(items: T[]): T | null
                 |
                 +-- T = Book  =>  Book | null
                 +-- T = User  =>  User | null
```

## أبسط مثال

```ts
function first<T>(items: T[]): T | null {
  return items[0] ?? null;
}

const title = first(['A', 'B']);
```

## شرح المثال

- `<T>` يعلن المعامل.
- `items: T[]` مصفوفة من ذلك النوع.
- الإرجاع `T | null`.
- في النداء، `T` يُستنتج `string` من المصفوفة. `title` نوعه `string | null`.
- `??` يحوّل `undefined` إلى `null` إن كانت المصفوفة فارغة، فيبقى الإرجاع صادقًا.

تحديد صريح: `first<string>(['A'])`. نادرًا تحتاجه إذا كان الاستنتاج واضحًا.

## ماذا يحدث داخليًا؟

الناتج دالة JavaScript واحدة بلا أثر لـ `T`. الفحص كله قبل التشغيل. لذلك `Promise<Book>` يعلمك أنت والمترجم، ووقت التشغيل هو Promise عادية قد تُرفض أو تُحل بأي قيمة إن كذبت على المترجم.

في Nest ستقرأ `Promise<Book>` على دوال المتحكم غير المتزامنة. الإطار ينتظر الـ Promise ثم يرسل القيمة. النوع لا ينتظر عنك؛ `async` يفعل.

## مثال عملي

نتيجة بحث تستخدمها خدمة:

```ts
interface Page<T> {
  items: T[];
  total: number;
}

function emptyPage<T>(): Page<T> {
  return { items: [], total: 0 };
}
```

`Page<Book>` و `Page<Review>` عقدان مختلفان للمترجم من قالب واحد. الترقيم لاحقًا سيعمم هذه الفكرة على الاستعلامات. القيد:

```ts
function idOf<T extends { id: string }>(entity: T): string {
  return entity.id;
}
```

`T extends ...` يشترط حقل `id`. كائن بلا `id` يُرفض.

## أخطاء شائعة

- `function parse<T>(text: string): T { return JSON.parse(text); }` تكذب: `T` لا يفحص JSON. الناتج يجب أن يكون `unknown` ثم فحصًا.
- DTO عام مثل `class Query<T>` مع توقع أن `ValidationPipe` يعرف حقول `T`. لا يعرف.
- قيود معقدة في أول خدمة. ابدأ بنوع ثابت، وأدخل `<T>` عندما تتكرر الدالة.

## متى نستخدمه؟

عندما تبقى علاقة بين مدخل وناتج عبر أكثر من نوع، كما في `Page<T>` و `Promise<T>` و `Repository<T>`.

## متى لا نستخدمه؟

لدالة تتعامل مع كتاب فقط. `Book` أوضح من `T extends Book`.

## مقارنة

| | نوع ثابت | Generic |
| --- | --- | --- |
| التكرار | دالة لكل نوع | دالة واحدة |
| وقت التشغيل | لا أثر زائد | لا أثر لـ T |
| ValidationPipe | على صنف ملموس | لا يستنتج T |

## تمارين

### تمرين سهل

ما نوع `first([1, 2])`؟

<details>
<summary>الحل</summary>

`number | null`.

</details>

### تمرين متوسط

اكتب `last<T>` تعيد آخر عنصر أو `null`.

<details>
<summary>الحل</summary>

```ts
function last<T>(items: T[]): T | null {
  return items[items.length - 1] ?? null;
}
```

</details>

### تمرين متقدم

لماذا توقيع `parse<T>(text: string): T` خطر على حدود HTTP؟

<details>
<summary>الحل</summary>

المستدعي يكتب `parse<Book>(body)` والمترجم يصدق، بينما `JSON.parse` يعيد قيمة غير مضمونة. تختفي العلاقة عند التشغيل. أرجع `unknown` وافحص، أو استخدم Pipe يعيش وقت التشغيل.

</details>

## أسئلة مراجعة

- ماذا يثبت `<T>` بين المدخل والناتج؟
- هل `T` موجود في JavaScript الناتج؟
- لماذا DTO العام لا يكفي لـ `ValidationPipe`؟

## الخلاصة

Generic يربط الأنواع في دالة واحدة من غير نسخه. هو فحص كتابة. حدود الشبكة تحتاج فحصًا حيًا، لا معامل نوع.

## ماذا بعد؟

كل استدعاء قاعدة بيانات في Nest تقريبًا `Promise`. الدرس التالي يثبت `async/await`.

## المصادر الرسمية

- TypeScript — Generics: https://www.typescriptlang.org/docs/handbook/2/generics.html
- NestJS — Validation: https://docs.nestjs.com/application/validation
