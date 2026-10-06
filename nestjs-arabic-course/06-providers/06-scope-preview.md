# Scope: نظرة أولى

## الهدف من هذا الدرس

تمييز عمر Provider: `DEFAULT` نسخة للتطبيق، `REQUEST` نسخة لكل طلب وتعدي السلسلة، `TRANSIENT` نسخة لكل موضع حقن.

## المشكلة التي يحلها

حقل `memberId` على Service افتراضية يخلط أعضاء المكتبة لأن النسخة واحدة. جعل كل الخدمات `REQUEST` يحل الخلط وينشئ شجرة كائنات لكل طلب. Scope يسمي العمر الذي تحتاجه فعلًا.

## الفكرة ببساطة

Singleton هو الافتراضي ويُنصح به. `Scope.REQUEST` يربط العمر بالطلب ثم تُجمع النسخة بعده، وأي مستهلك يحقنه يصبح `REQUEST` أيضًا. `Scope.TRANSIENT` يعطي كل مستهلك نسخته من غير أن يغيّر Scope المستهلك.

## التعريف التقني

| القيمة | العمر |
| --- | --- |
| `Scope.DEFAULT` | نسخة واحدة لعمر التطبيق. تُنشأ مع الإقلاع وتُدمَّر مع الإيقاف |
| `Scope.REQUEST` | نسخة لكل طلب وارد، ثم تصبح قابلة للجمع |
| `Scope.TRANSIENT` | نسخة مخصصة لكل من يحقن الرمز، تُنشأ مع ذلك المستهلك |

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class VisitContext {}
```

على الوصفة الطويلة: `scope: Scope.TRANSIENT` بجانب `provide`. `Scope` من `@nestjs/common`. التصريح الصريح بـ Singleton هو `Scope.DEFAULT`.

حقن الرمز `REQUEST` من `@nestjs/core` يجعل المزود مرتبطًا بالطلب أصلًا، حتى من غير إعادة كتابة `scope`. أي مزود يعتمد على مزود `REQUEST` يرث هذا العمر، وهذا السلوك لا يُلغى.

WebSocket gateway و Cron وبعض استراتيجيات Passport تبقى Singleton. لا تضع عليها `REQUEST`.

## Mental Model

```text
VisitContext  scope REQUEST
LoansService  تحقن VisitContext  -> تصبح REQUEST
LoansController تحقن LoansService -> تصبح REQUEST
FeeRates      لا تعتمد على السلسلة -> تبقى DEFAULT
```

`TRANSIENT` لا يصعد هكذا. `AuditService` الـ Singleton التي تحقن `StampClock` العابرة تأخذ ساعة خاصة عند إنشائها مرة واحدة، وتبقى `AuditService` نسخة تطبيق.

## أبسط مثال

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import type { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class VisitContext {
  constructor(@Inject(REQUEST) private readonly request: Request) {}

  memberId() {
    return this.request.header('x-member');
  }
}
```

## شرح المثال

- `@Inject(REQUEST)` يمرر طلب المنصة لهذه النسخة فقط.
- `import type` لـ `Request` مقبول لأنه نوع Express لا رمز تسجيل.
- `header` يقرأ رأس هذا الطلب. النسخة لن تخدم طلبًا ثانيًا.
- كتابة `@Injectable({ scope: Scope.REQUEST })` مع `REQUEST` توثيق ظاهر. التوثيق يقول إن إعادة تحديد Scope تُتجاهل لأن الرمز نفسه Request-scoped.

## ماذا يحدث داخليًا؟

سلسلة `REQUEST` تُبنى لكل طلب ثم تُهمل. Controller في أعلى السلسلة يُعاد إنشاؤه مع الخدمة. تحت حمل مرتفع هذا تخصيص وجمع مستمر، فيزيد زمن الاستجابة. التوثيق يطلب الإبقاء على Singleton إلا عندما يلزم العمر بالطلب فعلًا، ويذكر أن تصميمًا منضبطًا لا يتجاوز غالبًا زيادة صغيرة. المؤشر العملي هنا: كل سلف في سلسلة الحقن يُعاد إنشاؤه.

`TRANSIENT` ينشئ نسخة لكل موضع `constructor`، وقت إنشاء ذلك الموضع. مستهلكان Singleton يأخذان ساعتين مختلفتين، وكل ساعة تعيش عمر مستهلكها. هذا ليس «نسخة لكل استدعاء دالة» ولا «نسخة لكل طلب».

## مثال عملي

```ts
import { Injectable, Scope } from '@nestjs/common';
import { VisitContext } from './visit-context.js';

@Injectable()
export class LoansService {
  constructor(private readonly visit: VisitContext) {}

  open(isbn: string) {
    return { isbn, member: this.visit.memberId() };
  }
}

@Injectable({ scope: Scope.TRANSIENT })
export class StampClock {
  readonly at = new Date();
}
```

`LoansService` مكتوبة من غير Scope، لكن حقن `VisitContext` يجعلها `REQUEST`. `StampClock` عابرة: من يحقنها يحصل على تاريخ إنشاء نسخته هو.

مزوّد Durable الذي يعيد استخدام شجرة حسب مستأجر موجود في توثيق Injection scopes. خارج هذه النظرة الأولى.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| حفظ `memberId` على Service افتراضية | الطلب التالي يرى عضو الطلب السابق |
| `REQUEST` على عميل قاعدة البيانات المشترك | كل من يحقن العميل يصبح لكل طلب، والتكلفة تصعد |
| توقع أن `TRANSIENT` يغيّر عمر Controller | المستهلك Singleton يبقى Singleton ويحتفظ بنسخته العابرة |
| `REQUEST` على WebSocket gateway | البوابة يجب أن تبقى نسخة واحدة حول المقبس |

## متى نستخدمه؟

`DEFAULT` للحساب والوصول إلى مخزن مشترك. `REQUEST` عندما تكون حالة هذا الطلب جزءًا من هوية الكائن، مثل سياق العضو المأخوذ من الطلب. `TRANSIENT` عندما يحتاج كل مستهلك حالة بداية خاصة، مثل ساعة أو متتبع لا تريد مشاركته مع مستهلك ثانٍ.

## متى لا نستخدمه؟

`REQUEST` لقراءة الرأس ثم تمرير النص كوسيط دالة. تمرير `memberId` من Controller إلى دالة Singleton أبقي على الشجرة الافتراضية. متجر `AsyncLocalStorage` لكل طلب موثّق كبديل لاحق عندما تكثر الطلبات؛ لا تبنِه في هذا الدرس.

## مقارنة

| Scope | متى تُنشأ النسخة | هل يصعد إلى من يحقنها |
| --- | --- | --- |
| `DEFAULT` | مرة عند الإقلاع | النسخة تُشارك |
| `REQUEST` | لكل طلب | نعم، والسلسلة كلها لكل طلب |
| `TRANSIENT` | مع كل مستهلك | لا؛ المستهلك يبقي Scope |

## تمارين

### تمرين سهل

ما Scope صنف عليه `@Injectable()` من غير خيارات؟

<details><summary>الحل</summary>

`Scope.DEFAULT`، أي Singleton لعمر التطبيق.

</details>

### تمرين متوسط

`LoansController` يحقن `LoansService` التي تحقن `VisitContext` ذات `REQUEST`. ما عمر Controller؟

<details><summary>الحل</summary>

`REQUEST`. العدوى تصعد حتى Controller، فيُنشأ لكل طلب مع الخدمة والسياق.

</details>

### تمرين متقدم

`AuditService` افتراضية تحقن `StampClock` العابرة. كم نسخة ساعة تُنشأ لطلبين متتاليين، ولماذا؟

<details><summary>الحل</summary>

نسخة ساعة واحدة. `AuditService` تُنشأ مرة عند الإقلاع فتأخذ ساعتها العابرة حينها. `TRANSIENT` لا يعني نسخة لكل طلب ما دام المستهلك Singleton.

</details>

## أسئلة مراجعة

- لماذا تُوصف `REQUEST` بأنها مكلفة؟
- ماذا يرث مزود يحقن رمز `REQUEST`؟
- ما الفرق بين «لكل حقن» و«لكل طلب»؟

## الخلاصة

الافتراضي نسخة تطبيق واحدة. `REQUEST` نسخة لكل طلب وتسحب من يحقنها إلى العمر نفسه. `TRANSIENT` نسخة لكل موضع حقن من غير أن تغيّر عمر المستهلك.

## ماذا بعد؟

استبدال الصنف المرتبط بالرمز: Class Providers.

## المصادر الرسمية

- NestJS — Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
