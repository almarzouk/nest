# useExisting

## الهدف من هذا الدرس

إعطاء رمز ثانٍ للنسخة نفسها المسجّلة أصلًا، من غير بناء ثانٍ كما يفعل `useClass`.

## المشكلة التي يحلها

مكتبة تتوقع الرمز `'LOGGER'` ومستهلك قديم يحقن الصنف `LoggerService`. إن ربطت `useClass` مرتين حصلت على نسختين وحالة مزدوجة. `useExisting` لقب: الرمز الجديد يُحل إلى الرمز القديم، والنسخة واحدة ما دام النطاق Singleton.

## الفكرة ببساطة

```ts
{
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
}
```

مع تسجيل `LoggerService` في الوحدة نفسها. من يحقن النص ومن يحقن الصنف يريان المرجع نفسه.

## التعريف التقني

مثال التوثيق:

```ts
import { Injectable, Module } from '@nestjs/common';

@Injectable()
export class LoggerService {}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
  exports: [LoggerService, 'AliasedLoggerService'],
})
export class AppModule {}
```

الحقن:

```ts
constructor(
  private readonly primary: LoggerService,
  @Inject('AliasedLoggerService') private readonly alias: LoggerService,
) {}
```

في النطاق الافتراضي `primary === alias`.

`useExisting` يشير إلى رمز موجود. إن غاب `LoggerService` من التسجيل يفشل الحل. `useClass: LoggerService` تحت لقب نصي يبني نسخة ثانية حتى لو كان الصنف مسجّلًا أيضًا.

في `registerAsync` حقل `useExisting` يعني إعادة استخدام صنف خيارات موجود. الفكرة قريبة: لا تنسخ البناء. لا تخلط اسم الحقل في تكوين الوحدة مع مزود `useExisting` هذا، لكن القصد واحد.

صدّر اللقب إن كانت وحدة أخرى تحقنه. تصدير الصنف وحده لا يصدّر النص تلقائيًا.

مع `Scope.TRANSIENT` على الأصل، اللقب يتبع سلوك الرمز الأصلي عند حله. لا تستعمل اللقب لتغيير النطاق. النطاق يُعلن على التسجيل الذي يبني.

## Mental Model

```text
LoggerService        -> يبني نسخة أ
AliasedLoggerService -> يشير إلى الرمز -> النسخة أ
useClass تحت لقب     -> يبني نسخة ب
```

المساواة مرجع في Singleton، لا تشابه حقول.

## أبسط مثال

المقطعان أعلاه. أضف في اختبار:

```ts
expect(app.get(LoggerService)).toBe(app.get('AliasedLoggerService'));
```

`toBe` يتحقق من المرجع. إن كتب أحدهم `useClass` بدل `useExisting` يفشل التوقع، وهذا اختبار مفيد عند تغليف مكتبة.

## شرح المثال

ترتيب المصفوفة يسجل الأصل ثم اللقب. اللقب لا يستدعي constructor. حقول تُملأ على `primary` تظهر عبر `alias` لأنهما كائن واحد. لذلك اللقب لا يعزل حالة اختبار. للعزل استعمل `useValue` على الرمز الذي يحقنه المستهلك.

## ماذا يحدث داخليًا؟

1. الحاوية تسجل `LoggerService` كبناء صنف.
2. تسجل النص كإحالة إلى ذلك الرمز.
3. أول حقن لأي من الرمزين يبني مرة إن كان Singleton.
4. الحقن الثاني يعيد المخزون.
5. التصدير قائمة منفصلة لكل رمز تريد إخراجه.
6. دورة `useExisting` نحو رمز يحيل إلى نفسه حلقة إعداد. أشر إلى الرمز الذي يبني فعلًا.

## مثال عملي

عقد قديم `PaymentGateway` مجرد، والتنفيذ الحالي `StripeGateway` مسجّل لتطبيق واحد. مستهلك جديد يحقن الصنف المجرد ومستهلك قديم يحقن `'PAYMENT_GATEWAY'`.

```ts
{
  provide: 'PAYMENT_GATEWAY',
  useExisting: StripeGateway,
}
```

وسجّل `{ provide: PaymentGateway, useExisting: StripeGateway }` إن كان المجرد صالحًا كرمز لأنه صنف مجرد يبقى وقت التشغيل. نسخة Stripe واحدة تغلق في `onModuleDestroy` مرة. `useClass` تحت الرمزين كان سيغلق مرتين أو يترك عميلًا ثانيًا.

## أخطاء شائعة

- `useClass` ظنًا أنه لقب، فتزدوج النسخة.
- لقب بلا تسجيل الأصل.
- تصدير واحد ونسيان الرمز الذي تستعمله الوحدة الأخرى.
- توقع عزل حالة عبر اللقب.
- سلسلة `useExisting` دائرية.

## متى نستخدمه؟

رمز قديم ورمز جديد يجب أن يشيرا إلى Singleton واحد، أو اسم تتوقعه مكتبة خارجية.

## متى لا نستخدمه؟

تنفيذان متعايشان. لكل رمز `useClass` خاص. مزيف اختبار يستبدل السلوك: `useValue` على الرمز المستهلك.

## مقارنة

| | النسخ | يبني |
| --- | --- | --- |
| `useExisting` | واحدة للرموز المحالة في Singleton | لا |
| `useClass` مكرر | نسخة لكل تسجيل | نعم |
| `useValue` مرتين بالمرجع نفسه | واحدة إن مررت المرجع نفسه | لا |

## تمارين

### تمرين سهل

ماذا يساوي `app.get(LoggerService)` و `app.get('AliasedLoggerService')` في مثال التوثيق؟

<details>
<summary>الحل</summary>

المرجع نفسه، لأن اللقب `useExisting` والنطاق Singleton.

</details>

### تمرين متوسط

لماذا التصدير يذكر الرمزين؟

<details>
<summary>الحل</summary>

كل رمز يُطلب خارج الوحدة يحتاج أن يكون في `exports`. تصدير الصنف لا يصدّر النص تلقائيًا.

</details>

### تمرين متقدم

متى يضر اللقب إن أردت في الاختبار مسجّلًا لا يكتب إلى الشبكة؟

<details>
<summary>الحل</summary>

اللقب يشارك النسخة الحقيقية. غطِّ الرمز الذي يحقنه الكود تحت الاختبار بـ `useValue`، ولا تضف لقبًا إلى التنفيذ الشبكي.

</details>

## أسئلة مراجعة

- ما الفرق عن `useClass` للرمز نفسه؟
- هل اللقب يعزل الحقول؟
- ماذا يجب أن يكون مسجّلًا أولًا؟

## الخلاصة

`useExisting` اسم إضافي لنسخة الرمز الموجود. استعمله لمنع بناء مزدوج عندما يتعدد المستهلكون في تسمية الاعتماد نفسه.

## ماذا بعد؟

Tokens و Symbols: ماذا تضع في `provide` عندما لا يوجد صنف صالح كمفتاح.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
