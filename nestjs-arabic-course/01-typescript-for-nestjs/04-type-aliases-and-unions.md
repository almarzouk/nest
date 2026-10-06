# Type Aliases و Union Types

## الهدف من هذا الدرس

تسمية نوع، ووصف قيمة قد تكون أحد احتمالين، مثل كتاب موجود أو غياب.

## المشكلة التي يحلها

`find` أحيانًا ترجع الكتاب وأحيانًا لا تجد شيئًا. إذا كان النوع `Book` فقط، الكود يعامل `null` ككتاب وينكسر على `book.title`. الاتحاد يجبرك على التفريق.

## الفكرة ببساطة

`type` اسم لنوع. الإشارة `|` تعني "هذا أو ذاك".

## التعريف التقني

Type alias يسمي أي نوع: كائنًا، أو اتحادًا، أو دالة. Union type قيمة من أحد الأطراف. المترجم يضيّق النوع (narrowing) بعد فحص `typeof` أو مقارنة `null`.

## Mental Model

```text
Book | null
   |-- إن فحصت book !== null أصبح Book
   |-- قبل الفحص لا يسمح بقراءة title تحت strictNullChecks
```

## أبسط مثال

```ts
interface Book {
  id: string;
  title: string;
}

type BookId = string;
type FindResult = Book | null;

function titleOf(result: FindResult): string {
  if (result === null) {
    return 'missing';
  }
  return result.title;
}
```

## شرح المثال

- `BookId` اسم أوضح من تكرار `string` في توقيعات المعرّفات.
- `FindResult` يعلن الاحتمالين.
- داخل `if` يضيّق المترجم `result` إلى `null`، وبعدها إلى `Book`.
- قراءة `result.title` قبل الفحص خطأ في `strict`.

## ماذا يحدث داخليًا؟

الاتحاد فحص وقت ترجمة فقط. `null` وقت التشغيل قيمة عادية. التضييق يتحول إلى `if` كتبتها أنت، ويبقى في JavaScript. لا يُضاف فحص سحري.

الاتحاد المميّز (discriminated union) يستخدم حقلًا مشتركًا مثل `status: 'ok'` و `status: 'error'` حتى يضيّق المترجم بقية الحقول. ستراه في أشكال أخطاء لاحقًا.

## مثال عملي

دور مستخدم كما ستمر في المصادقة، من غير بناء نظام أدوار:

```ts
type Role = 'customer' | 'staff' | 'admin';

function canEditPrice(role: Role): boolean {
  return role === 'staff' || role === 'admin';
}
```

`'guest'` لا تُقبل في النداء. هذا أضيق من `string`. وقت التشغيل، نص قادم من JSON ما زال `string` حتى تفحص أنه أحد الثلاثة. النوع لا يغلّف الطلب.

## أخطاء شائعة

- `type Book = { ... }` و Interface بالمعنى نفسه ثم الحيرة. للكائنات البسيطة الاثنان يصلحان. اختر Interface لأشكال الكائنات القابلة للتمديد في هذه الدورة، و `type` للاتحادات والأسماء.
- اتحاد بلا فحص ثم استخدام حقل موجود في طرف واحد.
- تمثيل الغياب بسلسلة فارغة `''` بدل `null` أو نوع نتيجة واضح. الفراغ يختلط بعنوان فارغ فعلًا.

## متى نستخدم alias؟

عندما يتكرر اتحاد أو اسم دالة أو اسم معرّف في أكثر من توقيع.

## متى لا نستخدمه؟

عندما تحتاج القيمة أن تعيش وقت التشغيل مع وسوم تحقق. هناك صنف. Alias يُمحى مثل Interface.

## مقارنة

| | Interface | Type alias |
| --- | --- | --- |
| اتحاد `A \| B` | لا يعبّر عنه مباشرة كاتحاد قيم | نعم |
| تمديد أشكال | `extends` | تقاطع `&` وأنواع أخرى |
| وقت التشغيل | يُمحى | يُمحى |

## تمارين

### تمرين سهل

ما نوع متغير يمكن أن يكون `number` أو `string`؟

<details>
<summary>الحل</summary>

`number | string`.

</details>

### تمرين متوسط

اكتب نوع نتيجة لحفظ كتاب: إما `{ ok: true; id: string }` أو `{ ok: false; reason: string }`، ودالة تقرأ `id` فقط في حالة النجاح.

<details>
<summary>الحل</summary>

```ts
type SaveResult = { ok: true; id: string } | { ok: false; reason: string };

function savedId(result: SaveResult): string | null {
  if (!result.ok) return null;
  return result.id;
}
```

بعد `result.ok` يضيّق المترجم الطرف الناجح فيظهر `id`.

</details>

### تمرين متقدم

لماذا `Role` لا تمنع قاعدة البيانات من إرجاع نص `'superuser'` إذا غيّر أحد الصفوف يدويًا؟

<details>
<summary>الحل</summary>

لأن الاتحاد فحص على قيم مكتوبة في TypeScript. صف قاعدة البيانات يدخل كقيمة وقت تشغيل. تحتاج فحصًا أو قيدًا في قاعدة البيانات. النوع يمسك الأخطاء في الكود الذي يمر بالمترجم.

</details>

## أسئلة مراجعة

- ماذا يعني `|`؟
- متى يضيّق المترجم الاتحاد؟
- لماذا Alias لا يصلح مكان DTO مُتحقق منه؟

## الخلاصة

`type` يسمي نوعًا، والاتحاد يجبر التفريق بين الاحتمالات في الكود. الأثر وقت التشغيل هو الفحص الذي تكتبه أنت.

## ماذا بعد؟

حقول قد تغيب، وحقول لا تُعاد كتابتها: Optional و Readonly.

## المصادر الرسمية

- TypeScript — Unions: https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#union-types
- TypeScript — Narrowing: https://www.typescriptlang.org/docs/handbook/2/narrowing.html
