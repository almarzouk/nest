# Prisma: Transactions

## الهدف من هذا الدرس

حفظ عمليتين مترابطتين معًا أو التراجع عنهما معًا عبر `prisma.$transaction`.

## المشكلة التي يحلها

إعارة كتاب وتسجيل صف في `loans` خطوتان. إن نجحت الأولى وسقطت العملية قبل الثانية، الكتاب مربوط بعضو ولا يوجد سجل إعارة. العكس يترك سجلًا بلا تحديث للكتاب. طلبان متزامنان قد يعيران النسخة الأخيرة إن فحصت العدد خارج معاملة.

## الفكرة ببساطة

المعاملة حدود على القاعدة: إما تُثبت كل الكتابات داخلها أو تُلغى كلها. Prisma يوفّر مصفوفة عمليات متتابعة، أو دالة تفاعلية تستقبل عميلًا `tx`. الكتابة داخل الدالة يجب أن تمر من `tx` لا من `this.prisma`.

## التعريف التقني

من صفحة Nest:

- الكتابة المتداخلة التي تنشئ الأب والأبناء في استدعاء واحد تعمل داخل معاملة تلقائيًا.
- المصفوفة: `this.prisma.$transaction([query1, query2])` تنفّذ بالترتيب وتُرجع مصفوفة نتائج.
- الدالة: `this.prisma.$transaction(async (tx) => { ... })` تُثبَّت إن رجع الوعد، وتُلغى إن رمى.

الخيار الثاني للمعاملة قد يحمل `maxWait` و `timeout` و `isolationLevel`.

```ts
const [loans, total] = await this.prisma.$transaction([
  this.prisma.loan.findMany({ where: { returned: false }, take: 20 }),
  this.prisma.loan.count({ where: { returned: false } }),
]);
```

القراءة المزدوجة هنا لقطة واحدة. الإعارة التي تعتمد على شرط تُكتب بالدالة التفاعلية حتى تستطيع قراءة الصف ثم تحديثه قبل أن يثبّت غيرك.

استعلام عبر `this.prisma` داخل الدالة يعمل خارج المعاملة. هذا أسهل خطأ في الدرس.

## Mental Model

```text
$transaction(async tx => {
  tx.book.update(...)     داخل الحدود
  tx.loan.create(...)     داخل الحدود
  this.prisma.book...     خارج الحدود
})
 نجاح -> COMMIT
 رمي   -> ROLLBACK
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../prisma/prisma.service.js";

@Injectable()
export class LoansService {
  constructor(private readonly prisma: PrismaService) {}

  loanOut(bookId: number, memberId: number) {
    return this.prisma.$transaction(async (tx) => {
      const book = await tx.book.findUnique({ where: { id: bookId } });
      if (!book || book.memberId !== null) {
        throw new Error("Book is not available");
      }
      await tx.loan.create({ data: { bookId, memberId } });
      return tx.book.update({
        where: { id: bookId },
        data: { memberId },
      });
    });
  }
}
```

## شرح المثال

رمي الخطأ يلغي `create` إن حدث قبل التحديث، ويلغي الاثنين إن فشل التحديث. الفحص `memberId !== null` يمنع إعارة كتاب خارج الرف داخل هذه المعاملة. تحت ضغط تزامن عالٍ قد تحتاج قيدًا فريدًا جزئيًا أو عزلًا أعلى حتى لا يمر طلبان من الفحص معًا. المعاملة تقصّر النافذة، والقيد يغلقها.

حوّل `Error` العام إلى استثناء HTTP عند المتحكم، لا تترك الرسالة الداخلية للعميل كما هي إن كانت تكشف تفاصيل.

## ماذا يحدث داخليًا؟

Prisma يبدأ معاملة على الاتصال، وينفّذ استعلامات `tx` عليها، ثم `COMMIT` أو `ROLLBACK`. العمليات خارج `tx` تستخدم اتصالًا آخر من المجمع وقد تُثبَّت حتى لو أُلغيت المعاملة. لذلك القاعدة العملية: كل أثر جانبي في القاعدة داخل الدالة يمر بـ `tx`.

مهلة `timeout` تلغي معاملة طالت لأن قفلًا انتظر. لا تضع نداء HTTP بطيئًا داخل الدالة، لأنه يمسك المعاملة مفتوحة.

## مثال عملي

إعادة الكتاب: `tx.loan.update` يضع `returned: true` و `tx.book.update` يصفّر `memberId`. إن فشل أحدهما يبقى الكتاب مع المستعير والسجل غير معاد، أو العكس، وهذا أسوأ من فشل الطلب كاملًا. المصفوفة تكفي إن لم تحتج قراءة بينهما:

```ts
await this.prisma.$transaction([
  this.prisma.loan.update({ where: { id: loanId }, data: { returned: true } }),
  this.prisma.book.update({ where: { id: bookId }, data: { memberId: null } }),
]);
```

## أخطاء شائعة

- استخدام `this.prisma` داخل الدالة التفاعلية.
- معاملة حول استدعاء بريد أو HTTP.
- افتراض أن فحصًا قبل `$transaction` يكفي تحت طلبين متزامنين.
- ابتلاع الخطأ داخل الدالة وإرجاع `null` فيُثبَّت نصف العمل لأن الوعد نجح.

## متى نستخدمه؟

عندما يجب أن تنجح كتابتان معًا: إعارة وسجل، تحويل رصيد، حذف أب بعد أبناء.

## متى لا نستخدمه؟

لقراءة واحدة أو كتابة واحدة. `create` المفرد معاملة بحد ذاته على مستوى الجملة.

## مقارنة

| مصفوفة `$transaction` | دالة `async (tx)` |
| --- | --- |
| عمليات معروفة سلفًا | فرع بعد قراءة |
| ترتيب ثابت | يمكنك الرمي قبل الكتابة الثانية |
| لا عميل `tx` يدوي | كل استعلام على `tx` |

## تمارين

### تمرين سهل

ماذا يحدث لصف `loan` إن رمى `book.update` داخل الدالة بعد `loan.create` على `tx`؟

<details>
<summary>الحل</summary>

يُلغى مع المعاملة. لا يبقى سجل إعارة بلا تحديث كتاب.

</details>

### تمرين متوسط

لماذا نداء إرسال بريد داخل `$transaction` فكرة سيئة؟

<details>
<summary>الحل</summary>

البريد بطيء وقد ينجح ثم تُلغى المعاملة، أو يطيل القفل. أرسل البريد بعد نجاح المعاملة، أو سجّل نية الإرسال في الجدول نفسه داخل المعاملة.

</details>

### تمرين متقدم

طلبان يعيران الكتاب نفسه ويفحصان `memberId` قبل استدعاء `$transaction` لا داخله. أين الثغرة؟

<details>
<summary>الحل</summary>

الفحص خارج القفل. كلاهما يرى الكتاب متاحًا ثم يكتبان. انقل القراءة والكتابة إلى الدالة نفسها، وعضّد ذلك بقيد يمنع مستعيرين للنسخة الواحدة.

</details>

## أسئلة مراجعة

- ما الفرق بين نجاح الدالة ورميها؟
- أي عميل يُستخدم داخل الدالة التفاعلية؟
- متى تكفي مصفوفة العمليات؟

## الخلاصة

`$transaction` يجمع آثار القاعدة التي لا يجوز أن تُرى منفصلة. `tx` هو حد الكتابة، والرمي هو التراجع.

## ماذا بعد؟

[وضع Prisma داخل Module](./10-prisma-in-a-module.md) يصدّر الخدمة لميزات الكتب والإعارات.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- Prisma — Transactions: https://www.prisma.io/docs/orm/prisma-client/queries/transactions
