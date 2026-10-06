# Existing Providers

## الهدف من هذا الدرس

إنشاء اسم ثانٍ لنسخة موجودة بـ `useExisting`، بحيث يشير رمزان إلى الكائن نفسه.

## المشكلة التي يحلها

مكتبة قديمة تحقن `'DeskAudit'` ونص جديد يحقن `AuditLog`. `useClass: AuditLog` تحت الرمز القديم يبني نسخة ثانية بعدّاد مستقل. `useValue` يحتاج مرجعًا في وقت التحميل، قبل أن تنشئ الحاوية `AuditLog`. `useExisting` يؤجل الجلب إلى النسخة المسجّلة للرمز الآخر.

## الفكرة ببساطة

```ts
{
  provide: 'DeskAudit',
  useExisting: AuditLog,
}
```

مع Singleton، من يطلب `'DeskAudit'` ومن يطلب `AuditLog` يحصلان على النسخة نفسها.

## التعريف التقني

`useExisting` اسم بديل لرمز مسجّل. الرمز الهدف، هنا `AuditLog`، يبقى في `providers` باختصاره أو بوصفته. الاسم البديل لا يستدعي `new`.

الفرق عن `useClass`:

- `useClass: AuditLog` تحت رمز ثانٍ ينفذ بناءً ويخزن النتيجة في خانة الرمز الجديد.
- `useExisting: AuditLog` يقرأ خانة `AuditLog` ويعيد ما فيها.

الفرق عن `useValue`: القيمة في `useExisting` هي نتيجة حل الرمز الآخر، لا مرجعًا كتبته قبل الإقلاع.

إن كان الرمزان `REQUEST`، فالاسم البديل يتبع نسخة ذلك الطلب للرمز الأصلي ضمن قواعد Scope. التوثيق يذكر الحالة الشائعة: Singleton على الطرفين يعني كائنًا واحدًا. لا تستخدم الاسم البديل لتغيير العمر؛ Scope يُحدَّد على التسجيل الأصلي.

تصدير الاسم البديل يكون بتصدير رمزه: `exports: ['DeskAudit']`، ويحتاج المستورد أن تُصدَّر السلسلة التي ترى `AuditLog` أيضًا داخل الـ Module التي عرّفت `useExisting`.

## Mental Model

```text
AuditLog      -> نسخة A
DeskAudit     useExisting AuditLog -> A
مقارنة ===    ناجحة داخل Singleton
```

## أبسط مثال

```ts
import { Injectable, Module } from '@nestjs/common';

@Injectable()
export class AuditLog {
  private lines = 0;

  write(message: string) {
    this.lines += 1;
    return { lines: this.lines, message };
  }
}

const deskAuditAlias = {
  provide: 'DeskAudit',
  useExisting: AuditLog,
};

@Module({
  providers: [AuditLog, deskAuditAlias],
  exports: [AuditLog, 'DeskAudit'],
})
export class AuditModule {}
```

## شرح المثال

- `AuditLog` تسجيل عادي ينشئ النسخة.
- `'DeskAudit'` لا يملك حالة خاصة. `write` يزيد `lines` على النسخة الأصلية.
- تصدير الرمزين يتيح لمستهلك قديم ومستهلك جديد مشاركة العدّاد.
- حذف `AuditLog` من `providers` يترك `useExisting` مشيرًا إلى رمز غائب، فيفشل الحل.

## ماذا يحدث داخليًا؟

حل `'DeskAudit'` يطلب حل `AuditLog` أولًا. إن كانت النسخة في الذاكرة تُعاد. إن لم تكن، تُبنى مرة ثم تُعاد للرمز الأصلي وللاسم البديل. بناء ثانٍ لا يحدث لمجرد وجود الاسم البديل.

`useClass: AuditLog` في مكان `useExisting` كان سيخزن نسخة في خانة `'DeskAudit'` منفصلة عن خانة `AuditLog`. عدّاد `lines` يتفرع. هذا ما يفرقه الاختبار السريع: استدعِ `write` من الرمزين وقارن `lines`.

## مثال عملي

جسر لخدمة المكتب التي ما زالت تطلب النص القديم:

```ts
import { Inject, Injectable, Module } from '@nestjs/common';
import { AuditLog } from './audit-log.js';
import { AuditModule } from './audit.module.js';

@Injectable()
export class DeskService {
  constructor(
    @Inject('DeskAudit') private readonly audit: AuditLog,
    private readonly direct: AuditLog,
  ) {}

  note(message: string) {
    this.audit.write(message);
    return this.direct.write(message);
  }
}

@Module({
  imports: [AuditModule],
  providers: [DeskService],
})
export class DeskModule {}
```

بعد النداء، `lines` يساوي 2 لأن الاستدعائين يصبان في النسخة نفسها. نوع الحقل `AuditLog` وصف للكائن. الرمز الفعلي للاسم القديم هو النص داخل `@Inject`. الرمز `direct` يُستنتج من النوع لأن تسجيل الصنف موجود ومصدَّر.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `useClass` بهدف المشاركة | نسخة ثانية وحالة متفرعة |
| `useExisting` من غير تسجيل الرمز الأصلي | لا خانة تُعاد |
| توقع `new` عند كل حقن للاسم البديل | Singleton يعيد المرجع المخزن |
| تصدير `'DeskAudit'` من Module لا ترى `AuditLog` | الحل لا يجد الهدف |

## متى نستخدمه؟

لاسم متوافق مع مكتبة أو مع رمز نصي قديم، عندما يجب أن تبقى الحالة والنسخة واحدة. ولإتاحة الرمز نفسه عبر رمز صنف ورمز نص في الوقت نفسه.

## متى لا نستخدمه؟

عندما تريد نسختين متعمدتين من الصنف نفسه. سجّل الرمز مرتين بـ `useClass` تحت رمزين، واقبل انفصال الحالة. وعندما تكون القيمة جاهزة قبل الإقلاع. تلك `useValue`.

## مقارنة

| الوصفة تحت رمز ثانٍ | النسخة |
| --- | --- |
| `useExisting: AuditLog` | نسخة تسجيل `AuditLog` |
| `useClass: AuditLog` | بناء جديد في خانة الرمز الثاني |
| `useValue: readyLog` | المرجع الذي ملكته قبل الإقلاع |
| اختصار `[AuditLog]` فقط | رمز واحد بلا اسم بديل |

## تمارين

### تمرين سهل

ماذا يجب أن يبقى في `providers` بجانب كائن `useExisting`؟

<details><summary>الحل</summary>

تسجيل الرمز الأصلي، مثل `AuditLog`، لأنه الخانة التي يُعاد محتواها.

</details>

### تمرين متوسط

`write` من `'DeskAudit'` ثم من `AuditLog` جعل `lines` يساوي 1. أي وصفة رُبط بها الاسم القديم؟

<details><summary>الحل</summary>

`useClass` أو تسجيل ثانٍ بنى نسخة أخرى. `useExisting` كان سيجعل الاستدعاء الثاني يرى `lines === 2`.

</details>

### تمرين متقدم

لماذا لا يصلح `useValue: auditInstance` بديلًا عن `useExisting` إن كانت `AuditLog` تُنشأ داخل الحاوية؟

<details><summary>الحل</summary>

`useValue` يثبت المرجع عند تحميل الملف، قبل أن تبني الحاوية `AuditLog` وتحقن اعتمادياتها. `useExisting` ينتظر تلك النسخة. صنع `new AuditLog()` يدويًا لتمريره في `useValue` يتجاوز حل اعتماديات الصنف.

</details>

## أسئلة مراجعة

- ما الذي يُعاد عند حل رمز `useExisting`؟
- كيف يتفرع العدّاد مع `useClass` تحت اسم ثانٍ؟
- أي رمز تصدّر ليصل الاسم البديل إلى Module أخرى؟

## الخلاصة

`useExisting` اسم آخر لخانة مسجّلة، فالرموز تشارك النسخة ضمن Singleton. `useClass` تحت الرمز الجديد يبني نسخة أخرى وحالة أخرى.

## ماذا بعد؟

الرموز التي ليست أصنافًا: Injection Tokens.

## المصادر الرسمية

- NestJS — Alias providers: https://docs.nestjs.com/fundamentals/custom-providers#alias-providers-useexisting
