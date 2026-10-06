# Tokens و Symbols

## الهدف من هذا الدرس

اختيار رمز الحقن: صنف، أو سلسلة، أو `Symbol`، أو صنف مجرد، ولماذا الواجهة وحدها لا تكفي.

## المشكلة التي يحلها

`constructor(private logger: LoggerService)` يعمل عندما `LoggerService` قيمة وقت التشغيل. `interface Logger` تُمحى في JavaScript فلا يوجد مفتاح تبحث عنه الحاوية. سلسلة `'LOGGER'` تعمل وتصادم إن استعملتها مكتبة أخرى. `Symbol` هوية فريدة إن شارك الجميع المرجع نفسه. الصنف المجرد يجمع العقد والرمز في أثر واحد.

## الفكرة ببساطة

الرمز ما يوضع في `provide` وفي `@Inject` أو في نوع المعامل إن كان صنفًا. الواجهة للنوع فقط. صدّر الرمز من ملف واحد واستورده، لا تنشئ `Symbol('LOGGER')` مرتين فهما رمزان مختلفان.

## التعريف التقني

سلسلة، كما في التوثيق، وتُجمع في ثابت:

```ts
export const CONNECTION = 'CONNECTION';
```

```ts
constructor(@Inject(CONNECTION) private readonly connection: Connection) {}
```

`Symbol`:

```ts
export interface LoggerPort {
  log(message: string): void;
}

export const LOGGER_SERVICE = Symbol('LOGGER_SERVICE');
```

```ts
{
  provide: LOGGER_SERVICE,
  useClass: PinoLoggerService,
}
```

```ts
constructor(@Inject(LOGGER_SERVICE) private readonly logger: LoggerPort) {}
```

الواجهة `LoggerPort` تصف الشكل فقط. `Symbol` هو المفتاح. وصفه في `Symbol('...')` للتشخيص في السجلات لا للمساواة. المساواة بـ `===` على المرجع المصدَّر.

صنف مجرد يبقى دالة في التشغيل، فيصلح رمزًا ونوعًا:

```ts
export abstract class LoggerService {
  abstract log(message: string): void;
}
```

```ts
{
  provide: LoggerService,
  useClass: PinoLoggerService,
}
```

```ts
constructor(private readonly logger: LoggerService) {}
```

بلا `@Inject` لأن `design:paramtypes` يرى الصنف المجرد. التوثيق يفضّل هذا عندما أثر واحد يكفي، ويفضّل `Symbol` للمكتبات حتى لا تتصادم السلاسل، والواجهة عندما لا تحتاج حقنًا أصلًا.

تعداد TypeScript يصلح رمزًا أيضًا إن بقيت قيمته وقت التشغيل.

## Mental Model

```text
واجهة     -> نوع فقط، تُمحى
سلسلة     -> مفتاح سهل التصادم، ثابت مصدَّر
Symbol    -> مفتاح فريد بالمرجع المصدَّر
صنف / مجرد -> مفتاح + نوع، وحقن بلا @Inject
```

## أبسط مثال

`LOGGER_SERVICE` مع `PinoLoggerService`. وحدة الكتب تحقن الرمز وتستدعي `log`. اختبار يغطي `{ provide: LOGGER_SERVICE, useValue: { log() {} } }`. لا تستعمل سلسلة `'LOGGER_SERVICE'` في الاختبار و `Symbol` في التطبيق. الرمز يجب أن يتطابق بالمرجع.

## شرح المثال

ملف `logger.token.ts` يصدّر `Symbol` مرة. كل المستوردين يحصلون على المرجع ذاته تحت ESM ما دام المسار واحدًا. إنشاء الرمز داخل الدالة المصنع لكل وحدة ينتج مفاتيح لا تجدها الحاوية عند الحقن.

## ماذا يحدث داخليًا؟

1. `provide` يسجل المفتاح في خريطة الحاوية.
2. للمعامل ذي النوع الصنفي، metadata تحفظ الصنف كمفتاح.
3. للمعامل ذي الواجهة، metadata فارغة أو `Object`، لذلك `@Inject` إجباري.
4. `@Inject(symbol)` يخزن المرجع في metadata الحقن.
5. حل لاحق يبحث بالمساواة. `Symbol('X')` جديد لا يساوي المصدَّر.
6. التصدير من الوحدة يجب أن يذكر المفتاح نفسه: `exports: [LOGGER_SERVICE]`.

## مثال عملي

منفذ مستودع، والعقد واجهة، والرمز `Symbol` لأن الدومين لا يستورد Prisma:

```ts
export const BOOKS = Symbol('BOOKS');

export interface BooksPort {
  findTitle(id: string): Promise<string | null>;
}
```

```ts
{ provide: BOOKS, useClass: PrismaBooksAdapter }
```

خدمة التطبيق `@Inject(BOOKS) books: BooksPort`. المحوّل في وحدة بنية تحتية. هذا يمهّد لفصل المستودع. هنا المطلوب أن الرمز يعيش في ملف يراه الطرفان من غير أن تصبح الواجهة مفتاحًا.

## أخطاء شائعة

- `provide: 'BooksPort'` في ملف و `@Inject(Symbol('BooksPort'))` في آخر.
- إنشاء `Symbol` داخل `forRoot` كل مرة.
- توقع حقن الواجهة بلا رمز.
- سلسلة عامة `'CONFIG'` تتصادم مع مكتبة.
- نسيان تصدير `Symbol`.

## متى نستخدمه؟

`Symbol` لمكتبة أو لعقد بلا صنف. الصنف المجرد عندما تريد حقنًا مباشرًا. السلسلة لأمثلة صغيرة داخل تطبيق واحد مع ثابت مصدَّر.

## متى لا نستخدمه؟

لا تخترع رمزًا لصنف مسجّل أصلًا بالاختصار. ولا تضع البيانات السرية داخل وصف `Symbol`. الوصف يظهر في أدوات التصحيح.

## مقارنة

| المفتاح | يبقى وقت التشغيل | `@Inject` | تصادم |
| --- | --- | --- | --- |
| صنف مجرد | نعم | غير لازم | هوية الصنف |
| سلسلة | نعم | لازم | ممكن بالاسم |
| `Symbol` المصدَّر | نعم | لازم | نادر إن وُحّد المرجع |
| واجهة | لا | لا تصلح وحدها | — |

## تمارين

### تمرين سهل

لماذا لا يُحقن `interface BooksPort` مباشرة؟

<details>
<summary>الحل</summary>

لأنه يُمحى عند الترجمة ولا يوجد مفتاح في الحاوية.

</details>

### تمرين متوسط

متى `Symbol('A')` لا يجد المزود المسجّل بـ `Symbol('A')`؟

<details>
<summary>الحل</summary>

عندما يُنشأ كل واحد بنداء منفصل. الهوية مرجع الرمز لا النص. صدّر واحدًا واستورده.

</details>

### تمرين متقدم

متى تختار صنفًا مجردًا بدل `Symbol` لعقد المسجّل؟

<details>
<summary>الحل</summary>

عندما تريد أن يكون الأثر نفسه العقد ورمز الحقن، فتكتب المعامل بنوع الصنف المجرد بلا `@Inject`. `Symbol` أنسب إن كان يجب فصل المفتاح عن أي صنف في مكتبة مشتركة.

</details>

## أسئلة مراجعة

- ماذا يُصدَّر مع المزود النصي؟
- ما دور وصف `Symbol`؟
- أين تضع الواجهة بالنسبة للرمز؟

## الخلاصة

الرمز مفتاح تشغيل: صنف، سلسلة ثابتة، أو `Symbol` واحد مصدَّر. الواجهة تبقى للنوع بجانب المفتاح، لا بدلًا منه.

## ماذا بعد؟

Async Factories: المصنع الذي تُنتظر قيمته قبل أن يُحقن أحد.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
