# Parameter Properties

## الهدف من هذا الدرس

فك السطر `constructor(private readonly usersService: UsersService) {}` كلمة كلمة. هذا هو السطر الذي وعدت به المقدمة.

## المشكلة التي يحلها

الشكل المطول يكرر الاسم ثلاث مرات: حقل، ومعامل، وإسناد. التكرار يفتح اختلافًا في الإملاء. TypeScript يسمح بإعلان الخاصية من مكان المعامل.

والمشكلة المعمارية التي يخدمها السطر: الصنف يعلن ماذا يحتاج، ولا ينشئه بـ `new` داخل الدوال.

## الفكرة ببساطة

معامل constructor مسبوق بمعدّل وصول يصبح خاصية على الكائن، وتُسند إليه قيمة المعامل تلقائيًا.

## التعريف التقني

Parameter property ميزة TypeScript: في constructor، المعامل المعلّم بـ `public` أو `private` أو `protected` (ومعها `readonly` إن شئت) يصرّح بحقل صنف ويهيئه من الوسيط. الأنواع تُمسح، وMetadata التصميم يمكن أن تسجل نوع المعامل إذا اجتمعت Decorator على الصنف مع `emitDecoratorMetadata`.

## Mental Model

```text
وسيط قادم من new أو من الحاوية
        |
        v
constructor(private readonly usersService: UsersService)
        |
        +-- ينشئ this.usersService
        +-- يجعله private و readonly
        +-- يسند الوسيط إليه
        |
        v
this.usersService داخل دوال الصنف
```

## أبسط مثال

```ts
class UsersService {
  findName(): string {
    return 'Ada';
  }
}

class UsersController {
  constructor(private readonly usersService: UsersService) {}

  show(): string {
    return this.usersService.findName();
  }
}

const controller = new UsersController(new UsersService());
controller.show();
```

## شرح المثال

- `UsersService` صنف بسيط بلا حالة. في Nest سيكون `@Injectable()` ومسجّلًا. الوسوم مؤجلة حتى الدرس 14 و 15، والحاوية حتى فصل الحقن. هنا نرى اللغة فقط.
- `constructor(private readonly usersService: UsersService)` خمسة قرارات:
  1. `constructor` تهيئة عند إنشاء المتحكم.
  2. `usersService` اسم الوسيط واسم الخاصية.
  3. `private` الخاصية غير متاحة كـ `controller.usersService` من ملف آخر في فحص TypeScript.
  4. `readonly` دوال الصنف لا تعيد `this.usersService = ...`.
  5. `: UsersService` نوع الوسيط. من ينادي `new` يجب أن يمرر كائنًا متوافقًا.
- جسم الـ constructor فارغ `{}` لأن الإسناد تولّد عنك. الناتج يشبه الإسناد الصريح `this.usersService = usersService`.
- `show` تستخدم `this.usersService`. من غير الخاصية لما وُجد `this.usersService` بعد انتهاء الـ constructor.
- السطر الأخير ينشئ الخدمة ثم المتحكم. هذا ما ستوقفه Nest عنك: الحاوية ترى النوع وتنشئ الخدمة أولًا ثم تمررها. شرط ذلك التسجيل في Module ووجود Metadata، لا هذا السطر وحده.

الشكل المطول المكافئ لغويًا:

```ts
class UsersController {
  private readonly usersService: UsersService;

  constructor(usersService: UsersService) {
    this.usersService = usersService;
  }
}
```

اختر المختصر في كود Nest لأنه شكل التوثيق الرسمي في كل الأمثلة تقريبًا.

## ماذا يحدث داخليًا؟

المترجم يولد الحقل والإسناد في JavaScript. شيء قريب من:

```js
class UsersController {
  constructor(usersService) {
    this.usersService = usersService;
  }
}
```

لا `private` في الناتج. نوع `UsersService` ليس موجودًا كفحص شبكة.

عندما تضيف Decorator على الصنف وتفعّل إصدار Metadata، يضيف المترجم تصميم أنواع المعاملات: الموضع الأول نوعه `UsersService`. Nest تقرأ ذلك وتبحث في الحاوية عن مزوّد بهذا الرمز (الصنف نفسه مفتاحًا افتراضيًا). إذا لم توجد Metadata، المعامل بلا نوع محفوظ، والحقن بالاعتماد على النوع لا يعمل حتى تمرر `@Inject` صراحة. هذا سبب الدرس 15، لا تفصيل إضافي هنا.

`readonly` لا يظهر في JavaScript. هو رفض لإعادة التعيين أثناء الترجمة.

## مثال عملي

متحكم يحتاج خدمتين. الترتيب في المعاملات هو ترتيب الوسائط:

```ts
class OrdersController {
  constructor(
    private readonly ordersService: OrdersService,
    private readonly mailer: Mailer,
  ) {}
}
```

افترض صنفين `OrdersService` و `Mailer` موجودين. `new OrdersController(orders, mailer)` يملأ الحقلين. عكس الترتيب خطأ نوعي إذا اختلفت الأنواع. الحاوية لاحقًا تطابق بالاسم النوعي لا بترتيب يدوي في مكان النداء، لأنك لا تكتب النداء. الترتيب داخل constructor ما زال ترتيب الحقول.

اعتماد دائري: كل صنف يطلب الآخر في constructor. اللغة تسمح بالإعلان، والتشغيل قد يرى أحدهما غير مهيأ. Nest لها `forwardRef` في فصل لاحق. تجنب الدورة أولى من حفظ الأداة.

## أخطاء شائعة

- كتابة المعامل من غير معدّل: `constructor(usersService: UsersService) {}` ثم استخدام `this.usersService`. الخاصية غير موجودة. المعدّل هو الذي يصرح بها، أو تصرح أنت بالحقل في الشكل المطول.
- `new UsersService()` داخل constructor "حتى يعمل". تلغي إمكانية تمرير بديل في الاختبار وتعيد الارتباط.
- الظن أن `private readonly` يسجّل الصنف في حاوية Nest. اللغة تهيئ حقلًا فقط. التسجيل `@Module({ providers })` و `@Injectable` دروس الإطار.
- وضع منطق ثقيل في constructor: نداء شبكة أو فتح قاعدة. البناء يصير بطيئًا وصعب الاختبار. التهيئة المؤجلة لها خطافات دورة حياة لاحقًا (`OnModuleInit`).

## متى نستخدم المختصر؟

في كل constructor حقن في هذه الدورة، كما في التوثيق.

## متى نكتب الشكل المطول؟

عندما تحتاج جسم constructor يحسب شيئًا من الوسيط قبل الإسناد، أو عندما يكون الوسيط غير المخزَّن على الحقل. يمكنك خلط: معامل بخاصية، ومعامل عادي تستخدمه داخل الجسم فقط.

## مقارنة

| الكتابة | ماذا يتولد |
| --- | --- |
| `constructor(usersService: UsersService) {}` | وسيط بلا خاصية |
| `constructor(private readonly usersService: UsersService) {}` | خاصية + إسناد |
| `constructor(private readonly usersService: UsersService) {}` داخل Nest مع Metadata | الخاصية نفسها + قدرة الحاوية على معرفة النوع |

## تمارين

### تمرين سهل

في المثال الأول، من أين تأتي قيمة `this.usersService` داخل `show`؟

<details>
<summary>الحل</summary>

من الوسيط الذي مُرر إلى `new UsersController(...)`، وأُسند تلقائيًا إلى الخاصية بسبب `private readonly`.

</details>

### تمرين متوسط

أصلح الكود:

```ts
class Desk {
  constructor(books: BooksService) {}
  count(): number {
    return this.books.count();
  }
}
```

افترض `BooksService` فيها `count(): number`.

<details>
<summary>الحل</summary>

```ts
class Desk {
  constructor(private readonly books: BooksService) {}
  count(): number {
    return this.books.count();
  }
}
```

</details>

### تمرين متقدم

اختبار يريد `show()` من غير قاعدة بيانات. كيف يساعد السطر المختصر، وما الذي ما زال ينقص حتى تفعل Nest ذلك تلقائيًا؟

<details>
<summary>الحل</summary>

الاختبار يمرر كائنًا فيه `findName` يرجع نصًا ثابتًا إلى `new UsersController(fake)`. السطر لا ينشئ الخدمة بنفسه. الذي ينقص لأتمتة Nest: وسم الخدمة، وتسجيلها، وMetadata النوع، وحاوية تبني المتحكم. اللغة توفر مكان الحقن فقط.

</details>

## أسئلة مراجعة

- أي كلمة في السطر تصرح بالخاصية؟
- ماذا يفعل `readonly` هنا، وأين لا يظهر؟
- لماذا جسم constructor فارغ وليس ناقصًا؟
- هل السطر وحده يسجّل مزوّدًا في Nest؟

## الخلاصة

`constructor(private readonly usersService: UsersService)` يصرح بحقل خاص غير قابل لإعادة التعيين، ويسند إليه ما يُمرَّر عند الإنشاء، ويعلن النوع الذي ستقرأه الحاوية لاحقًا من Metadata. هو إعلان احتياج، لا نداء `new` داخلي.

## ماذا بعد؟

أحيانًا خدمة تبني على صنف آخر. الوراثة تشرح `extends` قبل أن تقابل أصنافًا جاهزة توسّعها، مثل استراتيجيات لاحقة.

## المصادر الرسمية

- TypeScript — Parameter properties: https://www.typescriptlang.org/docs/handbook/2/classes.html#parameter-properties
- NestJS — Providers: https://docs.nestjs.com/providers
- TypeScript — `emitDecoratorMetadata`: https://www.typescriptlang.org/tsconfig/emitDecoratorMetadata.html
