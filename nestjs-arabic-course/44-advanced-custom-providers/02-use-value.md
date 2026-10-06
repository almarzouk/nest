# useValue

## الهدف من هذا الدرس

تسجيل قيمة جاهزة خلف رمز: ثابت إعداد، كائن مكتبة بنيته أنت، أو مزيف يستبدل خدمة في الاختبار.

## المشكلة التي يحلها

ليست كل الاعتماديات أصنافًا يبنيها Nest. عنوان جاهز، أو عميل أنشأته في الاختبار، أو كائن `{ findAll: () => [] }` يغطي `BooksService`. `useClass` يستدعي `new` ويحل constructor. `useValue` يأخذ المرجع كما هو ويحقنه لمن يطلب الرمز.

## الفكرة ببساطة

```ts
{
  provide: BooksService,
  useValue: fakeBooks,
}
```

أي `constructor(private readonly books: BooksService)` يستلم `fakeBooks`. الشكل البنيوي في TypeScript يكفي إن وُجدت الدوال التي تُستدعى. لا خطاف `onModuleInit` على كائن عادي لم يمر بصنف مسجّل.

## التعريف التقني

من التوثيق، الاستبدال في الاختبار أو في وحدة تغطي التسجيل:

```ts
import { Module } from '@nestjs/common';
import { BooksService } from './books.service.js';

const fakeBooks = {
  findAll: () => [{ id: 'b1', title: 'مقدمة' }],
};

@Module({
  providers: [{ provide: BooksService, useValue: fakeBooks }],
  exports: [BooksService],
})
export class BooksTestModule {}
```

ولقيمة ليست صنف خدمة:

```ts
import { connection } from './connection.js';

@Module({
  providers: [{ provide: 'CONNECTION', useValue: connection }],
})
export class AppModule {}
```

الحقن: `@Inject('CONNECTION') connection: Connection`. الرمز النصي يحتاج `@Inject` لأن المعامل بلا نوع يبقى رمزًا وقت التشغيل.

`useValue` لا يمر على اعتماديات الكائن. إن كان `fakeBooks` يحتاج Prisma فأنت تبنيه في الاختبار قبل التسجيل، أو تنتقل إلى `useFactory`.

النطاق الافتراضي يشارك المرجع نفسه. تغيير حقل على المزيف يظهر لكل من حقنه في تلك الحاوية.

## Mental Model

```text
أنت تبني القيمة -> useValue -> الحاوية تخزن المرجع
المستهلك يطلب الرمز -> المرجع نفسه
```

لا `new` من الإطار لهذا الرمز.

## أبسط مثال

`fakeBooks` أعلاه داخل `Test.createTestingModule({ providers: [...] })`. المتحكم الذي يحقن `BooksService` يستدعي `findAll` ويرى عنصرًا واحدًا من غير قاعدة. لا تسجل `BooksService` مرة ثانية كصنف في المصفوفة نفسها فتتنافس التسجيلات. آخر تسجيل للرمز هو ما تعتمده الحاوية حسب ترتيب البناء، فاجعل التغطية مقصودة في وحدة الاختبار.

## شرح المثال

الكائن ليس `@Injectable`. الدالة `findAll` تطابق ما يناديه المتحكم في هذا الاختبار. أضف الدوال التي سيصل إليها المسار فقط. مزيف أوسع من الاستدعاء يخفي أن الاختبار لا يمر بتلك الدوال.

## ماذا يحدث داخليًا؟

1. الإطار يخزن المرجع في خريطة الرمز.
2. عند بناء المستهلك يمرر المرجع إلى المعامل.
3. لا يستدعي `onModuleInit` على الكائن العادي.
4. إن كان الرمز صنفًا، التغطية تستبدل بناء ذلك الصنف في هذه الحاوية.
5. التصدير يمرر الرمز كما في أي مزود.
6. كائن قابل للتغيير مشترك. للاختبارات المتوازية في العملية نفسها ابنِ مزيفًا لكل `TestingModule`.

## مثال عملي

خيارات ثابتة لاختبار مهلة:

```ts
export const MAIL_OPTIONS = Symbol('MAIL_OPTIONS');

@Module({
  providers: [
    { provide: MAIL_OPTIONS, useValue: { timeoutMs: 5, from: 'test@shop.test' } },
  ],
})
export class MailTestModule {}
```

الخدمة `@Inject(MAIL_OPTIONS)`. لا تقرأ البيئة داخل الاختبار. أغلق عميلًا حقيقيًا وضعته بـ `useValue` في `afterAll` لأن الخطاف لن يراه.

## أخطاء شائعة

- توقع حقن اعتماديات داخل كائن `useValue`.
- مزيف ناقص يرمي عند أول دالة لم تكتبها.
- مشاركة مزيف ذي حالة بين اختبارات.
- وضع سر إنتاج كـ `useValue` داخل مستودع.
- نسيان `@Inject` مع الرمز النصي أو `Symbol`.

## متى نستخدمه؟

ثابت، كائن مكتبة مبني مسبقًا، أو مزيف اختبار شكله يطابق الاستدعاء.

## متى لا نستخدمه؟

عندما يجب أن تبني الحاوية صنفًا وتعبئ constructor. استعمل `useClass`. عندما تتوقف القيمة على مزود آخر، استعمل `useFactory`.

## مقارنة

| | `useValue` | `useClass` |
| --- | --- | --- |
| البناء | خارج الحاوية | داخلها |
| خطافات الصنف | لا على الكائن العادي | نعم إن طبّقها الصنف |
| الاختبار | مزيف سريع | تنفيذ بديل كامل |
| الاعتماديات | تُجهَّز أنت | تُحل تلقائيًا |

## تمارين

### تمرين سهل

من ينفذ `new` لقيمة `useValue`؟

<details>
<summary>الحل</summary>

أنت قبل التسجيل، أو مكتبة أرجعت الكائن. الحاوية تخزن المرجع.

</details>

### تمرين متوسط

لماذا `@Inject('CONNECTION')` واجب؟

<details>
<summary>الحل</summary>

لأن الرمز ليس صنفًا يظهر في `design:paramtypes`. `@Inject` يمرر الرمز صراحة.

</details>

### تمرين متقدم

مزيف `useValue` فتح مقبسًا. لماذا لا يكفي `enableShutdownHooks` لإغلاقه؟

<details>
<summary>الحل</summary>

الكائن ليس صنف مزود يطبّق `OnModuleDestroy`. أغلق المقبس في الاختبار أو غلف القيمة بخدمة لها خطاف.

</details>

## أسئلة مراجعة

- ماذا يستلم المستهلك؟
- هل تُستدعى خطافات على الكائن؟
- متى تفضّل مصنعًا على القيمة؟

## الخلاصة

`useValue` يضع مرجعًا جاهزًا خلف الرمز. مناسب للثابت والمزيف، وبلا بناء ولا خطافات إلا إذا غلفته بصنف.

## ماذا بعد؟

`useFactory`: بناء القيمة من مزودات تُحقن في الدالة.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
