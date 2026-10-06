# Injection Tokens

## الهدف من هذا الدرس

تعريف رمز الحقن عندما لا يكون صنفًا: نص، `Symbol`، أو `enum`، وحقنه بـ `@Inject`.

## المشكلة التي يحلها

`constructor(private readonly flags: FeatureFlags)` يفشل إن كانت `FeatureFlags` واجهة، لأنها تُمسح ولا تترك `design:paramtypes`. جدول الأعلام قيمة تحت رمز لا يملك دالة بانية. الرمز المستقل يعطي الحاوية مفتاحًا يبقى وقت التشغيل.

## الفكرة ببساطة

المفتاح الافتراضي هو الصنف نفسه. عندما لا يوجد صنف، تختار نصًا أو `Symbol` أو `enum` وتضعه في `provide`. المستهلك يكرر المفتاح داخل `@Inject(token)`. الواجهة تبقى وصف TypeScript بجانب المعامل، والرمز هو ما تبحث عنه Nest.

## التعريف التقني

```ts
export const FEATURE_FLAGS = Symbol('FEATURE_FLAGS');

{
  provide: FEATURE_FLAGS,
  useValue: { reservations: true, loans: true },
}
```

```ts
constructor(@Inject(FEATURE_FLAGS) private readonly flags: FeatureFlags) {}
```

`FeatureFlags` واجهة. استيرادها `import type` صحيح لأنها ليست المفتاح. المفتاح `FEATURE_FLAGS` يُستورد استيراد قيمة حتى يبقى `Symbol` نفسه.

`Symbol('FEATURE_FLAGS')` الجديد في ملف آخر كائن مختلف. `===` بينهما كاذبة. صدّر الرمز من ملف واحد واستورده حيث التسجيل وحيث `@Inject`.

النص `'FEATURE_FLAGS'` يعمل، ويتصادم إن سجلت ميزة أخرى النص نفسه في سياق يرونه معًا. `Symbol` يفصل الهويات حتى لو تطابق الوصف. التوثيق يسمح أيضًا بـ `enum` كمفتاح.

الصنف المجرد يبقى وقت التشغيل، فيصلح مفتاحًا ووصفًا معًا:

```ts
export abstract class FeatureFlags {
  abstract enabled(name: string): boolean;
}
```

حينها `provide: FeatureFlags` والـ `constructor(private readonly flags: FeatureFlags)` من غير `@Inject`. استخدم النص أو `Symbol` عندما تريد المفتاح منفصلًا عن أي صنف. استخدم الصنف المجرد عندما تريد مفتاحًا ووصفًا في أثر واحد.

`@Optional()` يمكن أن يجاور `@Inject(token)` فيصبح الغياب `undefined`.

تصدير المفتاح كما هو: `exports: [FEATURE_FLAGS]`.

## Mental Model

```text
واجهة FeatureFlags     وصف للمترجم فقط
Symbol FEATURE_FLAGS   مفتاح السجل
useValue { ... }       القيمة تحت المفتاح
@Inject(FEATURE_FLAGS) يطلب المفتاح لا الواجهة
```

## أبسط مثال

```ts
import { Inject, Injectable, Module } from '@nestjs/common';

export interface FeatureFlags {
  reservations: boolean;
  loans: boolean;
}

export const FEATURE_FLAGS = Symbol('FEATURE_FLAGS');

const flagsProvider = {
  provide: FEATURE_FLAGS,
  useValue: { reservations: true, loans: false },
};

@Injectable()
export class DeskService {
  constructor(@Inject(FEATURE_FLAGS) private readonly flags: FeatureFlags) {}

  canReserve() {
    return this.flags.reservations;
  }
}

@Module({
  providers: [flagsProvider, DeskService],
  exports: [FEATURE_FLAGS],
})
export class DeskModule {}
```

## شرح المثال

- `@Inject` من `@nestjs/common` يثبت المفتاح في Metadata بدل النوع الممسوح.
- الواجهة لا تظهر في `providers`.
- `canReserve` تقرأ الحقل من كائن `useValue`.
- تصدير `Symbol` يتيح لـ Module أخرى حقن الأعلام من غير نسخ الكائن في `useValue` ثانٍ.

## ماذا يحدث داخليًا؟

`@Inject(token)` يكتب الرمز على المعامل. عند الحل تتجاهل Nest نوع TypeScript لذلك الموضع وتستخدم الرمز الصريح. للنص تقارن الحاوية النص. للـ `Symbol` تقارن هوية الكائن. لذلك يجب أن يكون `FEATURE_FLAGS` في ملف التسجيل هو المتغير نفسه في ملف الخدمة.

صنف مجرد مفتاحًا يعمل كأي دالة بانية موجودة بعد الترجمة، فيكفي `design:paramtypes` من غير `@Inject`.

ضع المفاتيح في ملف `flags.token.ts` عندما يكثر المستوردون، كما تقترح الوثائق لتجميع الثوابت.

## مثال عملي

`EnvFeatureFlags` صنف فيه `enabled`، ويُبنى تحت الرمز نفسه:

```ts
@Module({
  providers: [
    { provide: FEATURE_FLAGS, useClass: EnvFeatureFlags },
    DeskService,
  ],
})
export class FlagsModule {}
```

`DeskService` تبقى على `@Inject(FEATURE_FLAGS)`. اختبار يبدل الوصفة إلى `useValue` لكائن يوفّر الدوال نفسها، من غير تعديل الخدمة.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| حقن واجهة من غير `@Inject` | لا مفتاح وقت التشغيل |
| `Symbol('FEATURE_FLAGS')` جديد في كل ملف | مفتاحان لا يتطابقان |
| نص `'CONFIG'` في ميزتين داخل السياق نفسه | إحدى القيم تستبدل الأخرى أو يلتبس السجل |
| `import type` للمتغير `FEATURE_FLAGS` | الـ `Symbol` يُمسح ويضيع `@Inject` |

## متى نستخدمه؟

عندما تكون القيمة كائنًا أو مصنعًا بلا صنف تريد إظهاره في `constructor`. `Symbol` للمكتبات والتطبيقات الواسعة. النص لمفتاح محلي واضح في Module صغيرة. الصنف المجرد عندما تريد استنتاجًا بلا `@Inject`.

## متى لا نستخدمه؟

لصنف خدمة عادي مثل `FeesService`. الرمز هو الصنف، والاختصار `providers: [FeesService]` يكفي. ولا تختر `Symbol` ثم تنسخ تعريفه؛ متغير واحد مصدَّر هو العقد.

## مقارنة

| المفتاح | كيف يُحقن | ملاحظة |
| --- | --- | --- |
| صنف ملموس | نوع المعامل | الافتراضي |
| صنف مجرد | نوع المعامل | موجود وقت التشغيل |
| نص | `@Inject('NAME')` | قد يتصادم الاسم |
| `Symbol` مصدَّر | `@Inject(TOKEN)` | هوية فريدة ما دام المتغير واحدًا |
| واجهة | لا تصلح مفتاحًا | وصف فقط بجانب `@Inject` |

## تمارين

### تمرين سهل

لماذا لا تكفي `interface FeatureFlags` في `constructor` من غير `@Inject`؟

<details><summary>الحل</summary>

الواجهة تُحذف عند الترجمة ولا تظهر في `design:paramtypes`، فلا مفتاح تبحث عنه الحاوية.

</details>

### تمرين متوسط

ملف الخدمة أعاد `Symbol('FEATURE_FLAGS')` بدل استيراد الثابت. التسجيل يستورد الثابت من `flags.token.ts`. لماذا لا يجد الحل القيمة؟

<details><summary>الحل</summary>

كل نداء `Symbol` ينتج هوية جديدة. `@Inject` يبحث عن هوية مختلفة عن `provide`. استورد `FEATURE_FLAGS` من الملف الذي صدّره التسجيل.

</details>

### تمرين متقدم

متى تختار صنفًا مجردًا مفتاحًا بدل `Symbol`؟

<details><summary>الحل</summary>

عندما تريد أثرًا واحدًا يكون الوصف والمفتاح معًا، وتريد `constructor(flags: FeatureFlags)` من غير `@Inject`. اختر `Symbol` عندما يجب أن يبقى المفتاح مستقلًا عن شجرة الأصناف، خصوصًا في مكتبة قد تتصادم فيها أسماء النصوص.

</details>

## أسئلة مراجعة

- ما المفتاح الافتراضي لـ Provider صنف؟
- ماذا يستورد المستهلك استيراد قيمة: الواجهة أم `Symbol`؟
- كيف تصدّر مزود `useValue` مربوًا بـ `Symbol`؟

## الخلاصة

رمز الحقن مفتاح السجل. الصنف هو المفتاح الافتراضي، والنص والـ `Symbol` والـ `enum` مفاتيح عندما لا يوجد صنف. `@Inject` يسمي المفتاح، والواجهة تبقى وصفًا، ومتغير `Symbol` الواحد يُستورد في الطرفين.

## ماذا بعد؟

أمثلة تجمع التسجيل والحقن على ميزة مكتب الإعارات.

## المصادر الرسمية

- NestJS — Non-class-based tokens: https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens
- NestJS — Interfaces and abstract classes: https://docs.nestjs.com/fundamentals/custom-providers#interfaces-and-abstract-classes
