# Class Providers

## الهدف من هذا الدرس

استخدام `useClass` عندما يكون الرمز في `provide` صنفًا مختلفًا عن الصنف الذي يُنشأ.

## المشكلة التي يحلها

الاختصار `providers: [Notifier]` ينشئ `Notifier` دائمًا. بيئة التطوير تحتاج `ConsoleNotifier` والإنتاج `MailNotifier`، والمستهلك ما زال يطلب `Notifier` في `constructor`. `useClass` يفصل الرمز عن الصنف المبني.

## الفكرة ببساطة

```ts
{
  provide: Notifier,
  useClass: MailNotifier,
}
```

من يطلب `Notifier` يحصل على نسخة من `MailNotifier`. الحاوية تنادي البناء على `MailNotifier`، وتحل اعتماديات `MailNotifier` لا اعتماديات رمز قديم.

## التعريف التقني

`useClass` يختار الصنف وقت تقييم الملف، أي عند الإقلاع. شرط `NODE_ENV` في المثال العملي يُقرأ عند تحميل Module لا عند كل طلب. وضع الكائن في متغير ثم تمريره إلى `providers` تنظيم فقط.

الصنف المختار يستبدل تنفيذًا افتراضيًا للرمز نفسه إن وُجد في السياق. `MailNotifier` تحتاج `@Injectable()` إن كانت لها اعتماديات، وتلك الاعتماديات تُذكر في `providers`. الوصفة لا تسجلها تلقائيًا.

النطاق الافتراضي يخزن نسخة واحدة لكل رمز `provide`. تسجيل `MailNotifier` باختصار إضافي بجانب `{ provide: Notifier, useClass: MailNotifier }` يصنع رمزًا ثانيًا ونسخة ثانية. الاسمين مع النسخة نفسها يأتيان في `useExisting`.

## Mental Model

```text
constructor(private readonly notifier: Notifier)
        |
        v
الرمز Notifier
        |
        v
useClass: MailNotifier  ->  new MailNotifier(...)
```

## أبسط مثال

```ts
import { Injectable, Module } from '@nestjs/common';

@Injectable()
export class Notifier {
  send(_text: string) {
    return { channel: 'base' };
  }
}

@Injectable()
export class MailNotifier extends Notifier {
  override send(text: string) {
    return { channel: 'mail', text };
  }
}

@Module({
  providers: [{ provide: Notifier, useClass: MailNotifier }],
  exports: [Notifier],
})
export class NotifyModule {}
```

## شرح المثال

- المستهلك يستورد `Notifier` ويحقنه كالمعتاد. لا يعرف `MailNotifier`.
- الوراثة هنا عقد مجال صغير. صنفان متوافقان في الدوال يكفيان أيضًا؛ الحاوية لا تفرض `extends`.
- `exports: [Notifier]` يصدّر الرمز الذي يطلبه الآخرون، لا اسم الصنف المبني فقط.
- `_text` في القاعدة يوثّق التوقيع. التنفيذ الفعلي في `MailNotifier`.

## ماذا يحدث داخليًا؟

عند الحل ترى الحاوية وصفة `useClass`. تنظر إلى `MailNotifier` وتقرأ `design:paramtypes` لذلك الصنف، تبني الاعتماديات، ثم تنشئ النسخة وتخزنها تحت الرمز `Notifier`. طلب `MailNotifier` كرمز مستقل يفشل إن لم يكن له سجل خاص.

`NODE_ENV` يُقرأ عند تحميل Module. تغييره بعد الإقلاع لا يبدّل `useClass`. اختيار يتغير لكل طلب يحتاج `useFactory` أو Scope آخر، لا شرطًا مرة واحدة في الملف.

## مثال عملي

```ts
import { Injectable, Module } from '@nestjs/common';
import { ConsoleNotifier } from './console-notifier.js';
import { MailNotifier } from './mail-notifier.js';
import { Notifier } from './notifier.js';
import { DeskService } from './desk.service.js';

const notifierProvider = {
  provide: Notifier,
  useClass:
    process.env.NODE_ENV === 'development' ? ConsoleNotifier : MailNotifier,
};

@Module({
  providers: [notifierProvider, DeskService],
})
export class DeskModule {}
```

`DeskService` تطلب `Notifier` في البيئتين. Module الاختبار تمرر `{ provide: Notifier, useClass: FakeNotifier }` من غير تعديل `send`.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `providers: [MailNotifier]` والمُنشئ يطلب `Notifier` | الرمز المطلوب غير مسجّل |
| تسجيل `Notifier` و `MailNotifier` معًا بالإضافة إلى `useClass` | نسختان إن طُلب الرمزان |
| توقع تبديل `useClass` في كل طلب حسب رأس | الاختيار تم عند تحميل الملف |
| نسيان `@Injectable()` على الصنف البديل وفيه اعتماديات | فشل حل معاملات `MailNotifier` |

## متى نستخدمه؟

لاستبدال تنفيذ بآخر تحت رمز يعرفه المستهلك: بيئة، أو مزيف اختبار، أو تنفيذ عميل يطابق صنفًا أساسيًا.

## متى لا نستخدمه؟

عندما تكون القيمة جاهزة ولا تحتاج `new`: كائن إعداد أو عميل مُنشأ خارج Nest. ذلك `useValue`. وعندما يصنع الإنشاء خطوات تعتمد على مزودات أخرى. ذلك `useFactory`.

## مقارنة

| الوصفة | ماذا يُخزَّن تحت الرمز |
| --- | --- |
| اختصار `[Notifier]` | نسخة `Notifier` |
| `useClass: MailNotifier` | نسخة من الصنف المختار، تحت رمز `provide` |
| `useValue` | القيمة كما هي، من غير بناء صنف |
| `useExisting` | النسخة المسجّلة لرمز آخر، لا بناء جديد |

## تمارين

### تمرين سهل

في `{ provide: Notifier, useClass: MailNotifier }`، ماذا يكتب المستهلك في `constructor`؟

<details><summary>الحل</summary>

`Notifier`. هذا هو الرمز. `MailNotifier` هو الصنف الذي تبنيه الحاوية.

</details>

### تمرين متوسط

لماذا مزيف الاختبار يُمرَّر بـ `useClass` بدل تعديل `DeskService`؟

<details><summary>الحل</summary>

`send` تعتمد على الرمز. Module الاختبار تعيد ربط الرمز بـ `FakeNotifier`. الدالة تبقى على عقد `send`.

</details>

### تمرين متقدم

متى ينتج `useClass: MailNotifier` نسخة مختلفة عن اختصار `providers: [MailNotifier]` الموجود في الـ Module نفسها؟

<details><summary>الحل</summary>

عندما يكون `provide` رمزًا غير `MailNotifier`، مثل `Notifier`. لكل رمز خانة Singleton. الاختصار يملأ خانة `MailNotifier`، و `useClass` يملأ خانة `Notifier`. الخانتان نسختان إن طُلبتا معًا.

</details>

## أسئلة مراجعة

- ماذا يفصل `provide` عن `useClass`؟
- متى يُقيَّم شرط `NODE_ENV` في هذه الوصفة؟
- أي رمز تضعه في `exports` ليراه المستورد؟

## الخلاصة

`useClass` يبني صنفًا قد يختلف عن الرمز الذي يطلبه `constructor`. المستهلك يبقى على الرمز، والوصفة تختار التنفيذ عند الإقلاع.

## ماذا بعد؟

حقن قيمة جاهزة: Value Providers.

## المصادر الرسمية

- NestJS — Class providers: https://docs.nestjs.com/fundamentals/custom-providers#class-providers-useclass
