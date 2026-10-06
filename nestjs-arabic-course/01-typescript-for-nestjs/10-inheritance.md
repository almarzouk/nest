# Inheritance

## الهدف من هذا الدرس

فهم `extends` و `super` بما يكفي لقراءة أصناف توسع أساسًا جاهزًا، من غير بناء هرم وراثة في كل ميزة.

## المشكلة التي يحلها

سلوك مشترك بين أصناف، مثل "كل خطأ HTTP يحمل رمزًا ورسالة"، يمكن نسخه في كل ملف أو وضعه في أساس واحد. الوراثة أحد الحلول. التركيب (صنف يملك صنفًا آخر عبر constructor) هو الحل الذي تفضله Nest في الخدمات غالبًا.

## الفكرة ببساطة

`class AdminGuard extends GuardBase` يعني أن الابن يأخذ ما هو عام في الأب ويمكنه تجاوز دالة. `super` ينادي نسخة الأب.

## التعريف التقني

الوراثة في JavaScript سلسلة prototype. TypeScript يفحص أن الابن يحترم عقد الأب. دالة في الابن بنفس الاسم تستبدل دالة الأب عند الاستدعاء على كائن الابن. `super.method()` يصل إلى نسخة الأب.

## Mental Model

```text
Base: سلوك عام
  |
  +-- Child: يخصص جزءًا ويستدعي super عند الحاجة

التركيب البديل:
Child يملك Base كحقل محقون، ويناديه بالاسم
```

## أبسط مثال

```ts
class HttpError extends Error {
  constructor(
    readonly statusCode: number,
    message: string,
  ) {
    super(message);
  }
}

const missing = new HttpError(404, 'Book not found');
```

## شرح المثال

- `Error` أساس اللغة وفيه `message`.
- `super(message)` يجب أن يُستدعى قبل استخدام `this` في constructor الابن عندما يوجد constructor أب يحتاج تهيئة.
- `statusCode` خاصية عامة للقراءة عبر parameter property.
- `missing.message` قادم من الأب. `missing.statusCode` من الابن.
- Nest لاحقًا تعطي `HttpException` الجاهز. لا تبنِ هرم أخطاء قبل فصل Exception Filters. المثال لغة فقط.

## ماذا يحدث داخليًا؟

`new HttpError` يستدعي constructor الابن، وهو يستدعي constructor الأب عبر `super`، ثم يهيئ الحقول. `instanceof HttpError` و `instanceof Error` كلاهما صحيح. سلاسل `catch` تستطيع التقاط الأب والتفريق بالابن.

تجاوز دالة:

```ts
class Loud extends HttpError {
  constructor(message: string) {
    super(500, message.toUpperCase());
  }
}
```

الابن يجهّز الوسيط ثم يسلمه للأب.

## مثال عملي

في Passport لاحقًا سترى شكلًا مثل تمديد `PassportStrategy`. المعنى اللغوي واحد: صنف المكتبة أب، وصفك ابن يمرر خيارات إلى `super(...)` ويعرّف `validate`. لن نكتب الاستراتيجية هنا. اعرف فقط أن `super` في constructor ليس اختياريًا إذا كان الأب يحتاج وسائط.

خدمات العمل العادية في Nest: `OrdersService` لا ترث `BooksService` لتستخدم الكتب. تحقن `BooksService`. الوراثة بين الخدمات تربطهما إلى الأبد وتجعل الاختبار أثقل. الحقن يبقي العلاقة معلنة وقابلة للاستبدال.

## أخطاء شائعة

- نسيان `super()` في constructor الابن.
- استخدام `this` قبل `super()`.
- هرم `BaseService` ثم `CrudService` ثم `BookService` لنسخ دوال عامة. يتكرر القالب ويصعب الاستثناء. مزوّد صغير أوضح.
- تجاوز دالة وتغيير معناها بحيث كائن الابن لا يصلح مكان الأب. هذا كسر لتوقع الاستبدال (Liskov)، وسيعود في فصل SOLID.

## متى نستخدم الوراثة؟

لأخطاء متخصصة، أو عندما يفرض إطار أو مكتبة صنفًا أساسًا (`extends PassportStrategy` مثلًا في الوصفة الرسمية).

## متى نفضل التركيب؟

عندما "تستخدم" سلوكًا لا "تكون" نوعًا منه. الخدمة التي ترسل بريدًا تملك mailer محقونًا، وليست ابنًا لصنف البريد.

## مقارنة

| | Inheritance | Composition عبر constructor |
| --- | --- | --- |
| العلاقة | هو نوع من | يستخدم |
| في خدمات Nest | نادرة | الشكل الافتراضي |
| الاستبدال في الاختبار | أصعب إذا ثبت الأب | تمرر بديلًا للحقن |

## تمارين

### تمرين سهل

لماذا `missing instanceof Error` صحيح في المثال؟

<details>
<summary>الحل</summary>

لأن `HttpError` يمتد `Error`، فسلسلة prototype تصل إلى `Error`.

</details>

### تمرين متوسط

أصلح:

```ts
class NotFound extends HttpError {
  constructor() {
    this.kind = 'missing';
  }
}
```

<details>
<summary>الحل</summary>

```ts
class NotFound extends HttpError {
  readonly kind = 'missing';
  constructor() {
    super(404, 'Not found');
  }
}
```

`super` قبل أي استخدام لـ `this`. الحقل يمكن تهيئته في التصريح.

</details>

### تمرين متقدم

لماذا حقن `BooksService` داخل `OrdersService` أضعف ارتباطًا من `class OrdersService extends BooksService`؟

<details>
<summary>الحل</summary>

الحقن يعتمد على ما تستدعيه من دوال عامة ويمكن استبدال الكائن في الاختبار. الوراثة ترث كل التفاصيل المحمية وتجعل الطلب ابنًا للكتاب، وهذا وصف كاذب للمجال. تغيير أساس الكتب يهز الطلبات حتى في ما لا تستخدمه.

</details>

## أسئلة مراجعة

- ماذا يفعل `super` في constructor؟
- متى تفضّل الحقن على `extends` لخدمتين؟
- أي مكتبة لاحقًا ستجبرك على تمديد صنف؟

## الخلاصة

`extends` يشارك سلوك أب، و `super` يهيئه أو ينادي دالته. في Nest، الخدمات تتركب بالحقن. الوراثة لأساسات اللغة والمكتبات التي صممت لذلك.

## ماذا بعد؟

أساس لا يُنشأ وحده، والابن يملأ دالة ناقصة: الأصناف المجردة.

## المصادر الرسمية

- TypeScript — Inheritance: https://www.typescriptlang.org/docs/handbook/2/classes.html#inheritance
- NestJS — Passport (تمديد Strategy لاحقًا): https://docs.nestjs.com/recipes/passport
