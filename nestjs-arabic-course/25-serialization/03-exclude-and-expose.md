# Exclude و Expose

## الهدف من هذا الدرس

إخفاء `passwordHash` بـ `@Exclude`، وإظهار حقل محسوب بـ `@Expose`، مع ضبط `@Transform` حين يتغير شكل القيمة.

## المشكلة التي يحلها

العميل يحتاج `label` من `first` و `last` بلا إلزام القاعدة بعمود. ويحتاج ألا يرى التجزئة. مزخرف واحد على الكيان كله قد يخفي `id` إن عكست الاستراتيجية. `@Exclude` على السر أوضح من قائمة حذف متفرقة في المعالجات.

## الفكرة ببساطة

`@Exclude()` على الخاصية يسقطها من `instanceToPlain`. `@Expose()` على جالب يضيف قيمة تُحسب عند التسلسل. `@Expose({ name: "emailAddress" })` يغيّر اسم JSON. `@Transform` يستبدل القيمة، مثل أخذ `role.name` من كائن. الاستيراد من `class-transformer`. الاعتراض من الدرس السابق هو الذي يشغّل هذه المزخرفات.

## التعريف التقني

```ts
import { Exclude, Expose, Transform } from "class-transformer";

export class UserResponse {
  id: number;
  email: string;

  @Exclude()
  passwordHash?: string;

  @Expose()
  get label(): string {
    return this.email.split("@")[0];
  }

  @Transform(({ value }) => value?.code ?? value)
  role: string | { code: string };

  constructor(partial: Partial<UserResponse>) {
    Object.assign(this, partial);
  }
}
```

الصفحة تعرض `fullName` كجالب مع `@Expose`، و `@Transform(({ value }) => value.name)` على علاقة. المكتبة تبقي `role` سلسلة `reader` أو `editor`. إن وصلت العلاقة ككائن، التحويل يرسل الرمز فقط.

`@SerializeOptions({ excludePrefixes: ["_"] })` يستبعد البادئة لكل المسار. لا تغني عن `@Exclude` على التجزئة إن كان اسمها بلا بادئة.

استراتيجية الإظهار الحصري في `class-transformer` عبر `@Exclude()` على الصنف ثم `@Expose()` على كل حقل مسموح. ذلك قائمة سماح. مفيدة لصف مستخدم واسع. انسَ `@Expose` على `email` فيختفي البريد من JSON رغم أنك أردته. اخترها حين الأسرار أكثر من الحقول العامة، وأبقِ `@Exclude` الموضعي حين الحقول العامة هي الأغلب وسر واحد معروف.

`groups` في `@Expose({ groups: ["admin"] })` مع `@SerializeOptions({ groups: ["admin"] })` تُظهر حقلًا لمجموعة. لا تستعمل المجموعة كتفويض. المسار الإداري يضبط المجموعة بعد `@Can`. زائر يصل إلى المعالج يرى ما يرجعه المعالج.

مخطط Zod المقابل قائمة سماح بلا مزخرفات: الخصائص المعلنة فقط تخرج. `@Exclude` قائمة حذف. الحقل الجديد على الصف يخرج مع `@Exclude` الموضعي حتى تضيف مزخرفًا. مع المخطط أو مع `@Exclude` على الصنف لا يخرج حتى تذكره. هذا فرق عملي عند إضافة عمود سري لاحقًا.

## Mental Model

```text
@Exclude على السر يحذفه
@Expose على الجالب يضيف ناتجًا
@Transform يبدّل شكل الخاصية
عمود جديد بلا مزخرف قد يخرج إن كانت الاستراتيجية حذفًا فقط
```

## أبسط مثال

```ts
@Exclude()
passwordHash?: string;
```

## شرح المثال

سطر واحد على صنف الإخراج، مع الاعتراض، يغلق المسارات التي تعيد `UserResponse`. لا تضعه فقط على كيان TypeORM أو صنف Prisma المولَّد إن كان المعالج يعيد كائنًا من عميل Prisma. المزخرف يتبع الصنف الذي يُسلسَل.

الجالب `label` لا يُخزَّن. لا تجعله مدخلًا في DTO الإنشاء. `@Expose` للإخراج.

## ماذا يحدث داخليًا؟

`instanceToPlain` يرى البيانات الوصفية. الخاصية المستبعدة لا تُنسخ إلى الناتج. الجالب المعلَّم يُستدعى ويُكتب مفتاحه. `@Transform` يستلم `value` الحالية. إن كانت `role` أصلًا سلسلة، أعدها كما هي حتى لا ترمي عند `value.name`.

التحويل لا يشفّر. `@Transform` الذي يعيد التجزئة مقصوصة ما زال تسريبًا. احذفها.

## مثال عملي

`PostResponse` يستبعد أي `author.passwordHash` بتحويل المؤلف إلى `{ id, email }` عبر `@Transform`، أو بمنع تضمين المؤلف كاملًا. اختبار يتوقع `label` لـ `ada@library.test` أن يكون `ada`، وغياب `passwordHash`. أضف عمود `recoveryCode` في تمرين وانظر هل يخرج. إن خرج، الاستراتيجية الموضعية تحتاج مزخرفًا جديدًا أو انتقل إلى قائمة سماح.

## أخطاء شائعة

- `@Expose` على `passwordHash` لتوثيقه فيظهر في JSON.
- جالب بلا `@Expose` ثم توقع ظهور `label`. الصفحة تعلّم الجالب بالمزخرف.
- `@Transform` يرمي لأن القيمة أحيانًا سلسلة وأحيانًا كائن.
- مجموعة `admin` على مسار بلا حارس.

## متى نستخدمه؟

`@Exclude` لكل سر معروف على صنف الإخراج. `@Expose` لحقل مشتق. `@Transform` لشكل أبسط لعلاقة.

## متى لا نستخدمه؟

لتغيير معنى التفويض. إخفاء `role` في JSON لا يلغي الدور في السياسة. ولا لحذف السر من النسخ الاحتياطية للقاعدة.

## مقارنة

| `@Exclude` موضعي | `@Exclude` على الصنف |
| --- | --- |
| السر المسمى يختفي | لا يخرج إلا `@Expose` |
| عمود جديد قد يخرج | عمود جديد يختفي حتى تذكره |
| مناسب لسر واحد | مناسب لصف واسع |

## تمارين

### تمرين سهل

أي مزخرف يخفي `passwordHash`؟

<details>
<summary>الحل</summary>

`@Exclude()` من `class-transformer` على الخاصية، مع `ClassSerializerInterceptor`.

</details>

### تمرين متوسط

كيف تضيف `label` من غير عمود؟

<details>
<summary>الحل</summary>

جالب مع `@Expose()` يعيد القيمة المحسوبة.

</details>

### تمرين متقدم

أُضيف `sessionVersion` السري على الصف. لماذا خرج في JSON؟

<details>
<summary>الحل</summary>

الاستراتيجية تحذف المسمّى فقط. الحقل الجديد بلا `@Exclude`. ضع المزخرف أو انتقل إلى قائمة سماح أو مخطط Zod.

</details>

## أسئلة مراجعة

- من أين تُستورد المزخرفات؟
- ماذا يفعل `@Transform` في مثال الدور؟
- هل `groups` تستبدل `@Can`؟

## الخلاصة

`@Exclude` يحذف السر المعروف. `@Expose` يضيف المحسوب. عمود سري جديد يحتاج أن تدخله القاعدة، لا أن تأمل أن الاعتراض يخمنه.

## ماذا بعد؟

[Response DTOs](./04-response-dtos.md) صنف خروج منفصل عن صف القاعدة.

## المصادر الرسمية

- NestJS — Serialization: https://docs.nestjs.com/application/serialization
- class-transformer: https://github.com/typestack/class-transformer
