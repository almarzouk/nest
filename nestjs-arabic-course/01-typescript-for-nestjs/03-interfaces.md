# Interfaces

## الهدف من هذا الدرس

وصف شكل كائن يتكرر بين خدمة ومتحكم، ومعرفة أن Interface تُمحى عند التشغيل.

## المشكلة التي يحلها

كتابة `{ title: string; price: number }` في كل دالة تكرر العقد وتفتح باب اختلاف حرف. نريد اسمًا واحدًا للشكل.

## الفكرة ببساطة

`interface` اسم لشكل كائن: أي حقول، وبأي أنواع. المترجم يقارن الكائنات بهذا الشكل. وقت التشغيل لا توجد Interface.

## التعريف التقني

Interface في TypeScript تصف شكلًا هيكليًا. أي كائن فيه الحقول المطلوبة يصلح، حتى لو لم يقل `implements`. يمكن تمديدها بـ `extends`. لا تنتج كود JavaScript يتحقق من القيم.

## Mental Model

```text
وقت الكتابة:  Book شكل معروف
وقت التشغيل:  كائن عادي، والفحص من Validation لا من Interface
```

## أبسط مثال

```ts
interface Book {
  id: string;
  title: string;
  price: number;
}

function titleOf(book: Book): string {
  return book.title;
}
```

## شرح المثال

- `Book` عقد: ثلاثة حقول.
- `titleOf` تقبل أي كائن يطابق الشكل.
- `{ id: '2', title: 'Refactoring', price: 40 }` يصلح حتى لو لم يُنشأ بصنف `Book`.
- كائن ينقصه `price` يُرفض عند الفحص.

## ماذا يحدث داخليًا؟

بعد الترجمة تختفي `interface Book` بالكامل. لا دالة `instanceof Book`. لذلك توثيق Validation في Nest يقول: لا تُصدر Metadata للـ generics أو الـ interfaces، و `ValidationPipe` لا يتحقق من DTO مُعلَن كـ interface. استخدم صنفًا ملموسًا. هذا الفرق سيُعاد في فصل DTO، وهو سبب أن عقد الشبكة صنف لا Interface.

## مثال عملي

عقد قراءة منفصل عن عقد الإنشاء، لأن العميل لا يرسل `id`:

```ts
interface CreateBook {
  title: string;
  price: number;
}

interface Book extends CreateBook {
  id: string;
}
```

`extends` يضيف الحقول. `Book` فيه العنوان والسعر والمعرّف. الدالة التي تحفظ تأخذ `CreateBook` وتعيد `Book`. هذا تمييز مدخلات ومخرجات ستسميه الدورة لاحقًا Input DTO و Response، عندما تصبح أصنافًا.

## أخطاء شائعة

- استخدام Interface كـ DTO تتحقق منه `ValidationPipe`. الفحص لن يرى الحقول.
- `import type` مفهوم لاحقًا: النوع الممحو لا يُستورد كقيمة. الصنف الذي تحتاجه Nest وقت التشغيل يُستورد استيراد قيمة.
- الظن أن `implements` يفحص البيانات القادمة من JSON.

## متى نستخدم Interface؟

لشكل داخلي بين دوال TypeScript: ناتج خدمة، أو إعدادات، أو عقد لا يُفحص بـ class-validator.

## متى لا نستخدمها؟

كجسم `@Body()` الذي يجب أن يُتحقق منه وقت التشغيل عبر `ValidationPipe`. هناك صنف. `StandardSchemaValidationPipe` مسار آخر بمخطط، وسيأتي في التحقق، وهو أيضًا ليس Interface.

## مقارنة

| | Interface | Class |
| --- | --- | --- |
| وقت التشغيل | غير موجودة | موجودة كدالة بناء |
| `ValidationPipe` | لا يراها | يراها إذا وُسمت حقولها |
| حقن Nest | لا تُحقن | تُحقن إذا سُجّلت |

## تمارين

### تمرين سهل

هل `{ id: '1', title: 'A', price: 1, extra: true }` يطابق `Book` في المثال؟

<details>
<summary>الحل</summary>

نعم من جهة الأنواع: الحقول المطلوبة موجودة، والحقل الزائد مسموح عند تمرير كائن بهذا الشكل في أغلب سياقات التعيين المباشرة للكائن المعرّف كمتغير. التشدد على الحقول الزائدة (excess property check) يظهر أقوى عند تمرير الكائن حرفيًا في مكان ينتظر `Book`. الفكرة: Interface تفرض وجود المطلوب، ولا تحذف الزائد وقت التشغيل.

</details>

### تمرين متوسط

لماذا لا نضع `@IsString()` على Interface؟

<details>
<summary>الحل</summary>

الوسوم تحتاج هدفًا يعيش وقت التشغيل، وهو خاصية على صنف. Interface تُحذف، فلا مكان يلصق عليه الوسوم ولا Metadata يقرأها `ValidationPipe`.

</details>

### تمرين متقدم

خدمة تعيد `Promise<Book | null>`. الواجهة Interface. ماذا يبقى منها في JavaScript الناتج؟

<details>
<summary>الحل</summary>

لا شيء من اسم `Book`. تبقى الدالة التي تُرجع Promise وقيمة كائن أو `null`. من يستقبل الناتج يحتاج فحص `null` في الكود، لا اعتمادًا على أن Interface "ستمنع" الغياب وقت التشغيل.

</details>

## أسئلة مراجعة

- ماذا تصف Interface؟
- لماذا لا تصلح وحدها كـ DTO لـ `ValidationPipe`؟
- ما معنى أن النظام الهيكلي يقبل كائنًا لم يقل `implements`؟

## الخلاصة

Interface اسم لشكل أثناء الكتابة. شبكة Nest تحتاج صنفًا أو مخططًا يعيش بعد الترجمة عندما نريد فحصًا.

## ماذا بعد؟

الاتحاد والأسماء البديلة يصفان قيمة قد تكون أحد عدة أشكال.

## المصادر الرسمية

- TypeScript — Interfaces: https://www.typescriptlang.org/docs/handbook/2/everyday-types.html
- NestJS — Validation (interfaces لا تُفحص): https://docs.nestjs.com/application/validation
