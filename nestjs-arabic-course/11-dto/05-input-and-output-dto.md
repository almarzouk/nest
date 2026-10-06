# Input DTO و Output DTO

## الهدف من هذا الدرس

فصل صنف ما يرسله العميل عن صنف ما يعيده الخادم.

## المشكلة التي يحلها

`return member` بعد `save` يرسل `passwordHash` و `internalNote` لأنهما على الكيان. عقد الإنشاء الذي يُعاد كما هو يخفي `id` الذي ولّده الخادم أو يسربه إن وضعت أسرارًا على صنف الدخل.

## الفكرة ببساطة

Input DTO حقول مسموح أن تأتي من الخارج. Output DTO حقول مسموح أن تخرج على السلك. الكيان في الوسط قد يملك أكثر من الاثنين. المتحكم يترجم بينهما.

## التعريف التقني

لا واجهة في Nest اسمها InputDto. التسمية قرار تصميم: `CreateMemberDto` للداخل، `MemberResponseDto` للخارج. `ValidationPipe` يفحص الداخل. الخارج لا يمر على ذلك الأنبوب إلا إذا أعدت قراءته كطلب.

إرجاع الصنف الخارج من المتحكم يجعل شكل JSON قرارًا ظاهرًا في الملف. إرجاع الكيان يربط العميل بالأعمدة.

`ClassSerializerInterceptor` و `@Exclude()` أداة لاحقة في فصل التسلسل. هنا نبني كائن الرد في المتحكم أو في دالة `toResponse` حتى ترى الفرق من غير حزمة إضافية.

الحقول التي يولّدها الخادم، `id` و `joinedAt`، تظهر في الخارج وتغيب عن الداخل. الحقول التي يملكها العميل، `password` في طلب التسجيل، تظهر في الداخل وتغيب عن الخارج. الخارج قد يحمل `email` ولا يحمل كلمة المرور ولا تجزئتها.

## Mental Model

```text
CreateMemberDto     password, email, fullName
Member entity       + id, passwordHash, joinedAt, note
MemberResponseDto   id, email, fullName, joinedAt
```

سهمان، لا كائن واحد يسافر في الاتجاهين.

## أبسط مثال

```ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateMemberDto {
  @IsString()
  fullName: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

```ts
export class MemberResponseDto {
  id: string;
  fullName: string;
  email: string;
  joinedAt: string;
}
```

```ts
@Post()
create(@Body() dto: CreateMemberDto): MemberResponseDto {
  const saved = this.members.create(dto);
  return {
    id: saved.id,
    fullName: saved.fullName,
    email: saved.email,
    joinedAt: saved.joinedAt,
  };
}
```

## شرح المثال

- `password` على الدخل فقط. الخدمة تحوّله إلى تجزئة وتخزن التجزئة على الكيان.
- الرد يعدد الحقول. إضافة عمود `note` على الكيان لا تظهر للعميل إلا إذا أضفتها إلى `MemberResponseDto`.
- نوع الإرجاع يوثّق العقد للمترجم. وقت التشغيل ما يُسلسَل هو الكائن الذي رجعته.
- `CreateMemberDto` عليه زينة لأنه يُفحص. صنف الرد في هذا الدرس شكل مسمّى. إخفاء الحقول بالزينة درس `class-transformer` ثم فصل التسلسل.

## ماذا يحدث داخليًا؟

`ValidationPipe` يرى `CreateMemberDto` فقط. بعد النجاح تستلم الخدمة الدخل وتبني كيانًا. المتحكم ينتقي حقول الرد. Nest يسلسل القيمة المرجعة إلى JSON برمز `201` لـ `POST` ما لم تغيّر الرمز. التسلسل لا يحذف `passwordHash` إن كان المفتاح على الكائن المرجع.

لذلك الحذف قرار قبل `return`. دالة `toResponse(member): MemberResponseDto` في مكان واحد أفضل من نسخ الكائن في كل مسار.

قائمة الأعضاء تستخدم الشكل نفسه أو صنفًا أضيق مثل `MemberListItemDto` إن كانت القائمة بلا `joinedAt`. لا تُعد استخدام صنف الإنشاء كعنصر قائمة.

## مثال عملي

`POST /members` بجسم فيه كلمة مرور يرجع `201` و `{ id, fullName, email, joinedAt }`. اختبار العقد يتأكد أن المفتاح `password` و `passwordHash` غائبان عن JSON. `GET /members/:id` يرجع `MemberResponseDto` أيضًا. `PATCH` يأخذ `UpdateMemberDto` المشتق من الإنشاء بلا `password` إن كان تغيير السر مسارًا آخر. ثلاثة عقود، كيان واحد.

## أخطاء شائعة

- `return saved` من المستودع.
- وضع `passwordHash` على `MemberResponseDto` «لأن الواجهة الإدارية تحتاجه» في مسار عام.
- فحص صنف الرد بـ `ValidationPipe` على `@Body()`. الرد ليس جسم طلب.
- جعل الدخل والخرج صنفًا واحدًا فيه كل الحقول الاختيارية. العميل يرسل `id` ويظن أنه يحدّث مفتاح الصف.

## متى نستخدمه؟

كلما اختلف ما يُكتَب عما يُقرَأ: أسرار، حقول مولَّدة، ملاحظات داخلية، أشكال قائمة أقصر من شكل العنصر.

## متى لا نستخدمه؟

لمسار يعيد `{ ok: true }` بلا كيان. كائن صغير موضعي يكفي. ولمعامل مسار رقمي. ذلك ليس DTO جسم.

## مقارنة

| | Input | Output | كيان |
| --- | --- | --- | --- |
| الاتجاه | من العميل | إلى العميل | داخل العملية |
| `password` | نعم في التسجيل | لا | لا، توجد تجزئة |
| `id` | لا عند الإنشاء | نعم | نعم |
| من يفحصه | `ValidationPipe` | اختبار العقد | قاعدة البيانات |

## تمارين

### تمرين سهل

في أي صنف تعيش `password`، وفي أي صنف تعيش `joinedAt`؟

<details>
<summary>الحل</summary>

`password` في `CreateMemberDto`. `joinedAt` في `MemberResponseDto` وعلى الكيان، لا في عقد الإنشاء.

</details>

### تمرين متوسط

لماذا يرجع الاختبار فشلًا حين يظهر `passwordHash` في JSON رغم أن نوع الدالة `MemberResponseDto`؟

<details>
<summary>الحل</summary>

النوع وقت ترجمة. التسلسل يكتب مفاتيح الكائن الفعلي. الكيان رُجع كما هو. ابنِ كائن الرد بالحقول الأربعة فقط.

</details>

### تمرين متقدم

القائمة تحتاج `id` و `fullName`، والعنصر يحتاج أيضًا `email` و `joinedAt`. كيف تتجنب صنفًا واحدًا منتفخًا وصنفين بلا علاقة؟

<details>
<summary>الحل</summary>

`MemberListItemDto` بالحقلين، و `MemberResponseDto` يمتد منه أو يُبنى بـ `IntersectionType` / `PickType` من صنف رد أوسع. الدخل يبقى `CreateMemberDto` ولا يُستخدم في القائمة.

</details>

## أسئلة مراجعة

- لماذا لا يساوي الكيان عقد الخارج؟
- من يفحص دخل التسجيل؟
- ماذا يتسلسل: النوع أم الكائن المرجع؟
- أين تُخزَّن التجزئة؟

## الخلاصة

عقد الداخل وعقد الخارج صنفان. الكيان قد يكون أوسع من كليهما. المتحكم يختار ما يخرج حتى لا يسافر السر مع `return`.

## ماذا بعد؟

الدرس التالي زينة `class-validator` على عقد الدخل: القواعد التي يقرأها الأنبوب.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
- [Serialization](https://docs.nestjs.com/techniques/serialization)
