# DTO مقابل Interface

## الهدف من هذا الدرس

اختيار صنف لعقد يقرأه `ValidationPipe`، وفهم أين تبقى الواجهة مفيدة.

## المشكلة التي يحلها

`interface MemberInput { email: string }` تُرضي المترجم وتختفي في JavaScript. الأنبوب يبحث عن صنف وقت التشغيل فلا يجد زينة ولا `metatype` قابلًا للفحص.

## الفكرة ببساطة

الواجهة عقد مترجم. الصنف عقد مترجم وقيمة موجودة بعد الترجمة. فحص الدخل يحتاج الثانية. الواجهة تصلح لمخرجات داخلية لا تمر على الأنبوب.

## التعريف التقني

توثيق التحقق يقول: TypeScript لا يصدر بيانات وصفية للواجهات ولا للأنواع العامة، لذلك `ValidationPipe` لا يفحص DTO مُعلَنًا بها. استخدم أصنافًا ملموسة.

وكذلك: لا تستورد DTO باستيراد نوع فقط. `import type { CreateMemberDto }` يُمسح. اكتب `import { CreateMemberDto }`.

الواجهة ما زالت أداة صحيحة لشكل لا يُفحص بهذا الأنبوب: ناتج دالة خاصة، أو كائن تجمعه داخل الخدمة. هي لا توضع على `@Body()` إن كان الأنبوب العام هو خط الدفاع.

`implements SomeInterface` على الصنف ممكن حتى يطابق شكلًا داخليًا، لكن الزينة والفحص على الصنف.

## Mental Model

```text
interface  →  تختفي بعد tsc  →  الأنبوب لا يراها
class      →  تبقى دالة بناء  →  metatype يصل إلى ValidationPipe
import type →  يُمسح حتى لو كان الهدف صنفًا
```

## أبسط مثال

```ts
export interface MemberView {
  id: string;
  email: string;
}
```

```ts
import { IsEmail } from 'class-validator';

export class CreateMemberDto {
  @IsEmail()
  email: string;
}
```

```ts
import { Body, Controller, Post } from '@nestjs/common';
import { CreateMemberDto } from './create-member.dto.js';

@Controller('members')
export class MembersController {
  @Post()
  create(@Body() dto: CreateMemberDto): MemberView {
    return { id: 'm1', email: dto.email };
  }
}
```

`CreateMemberDto` يُفحص. `MemberView` شكل الإرجاع للمترجم فقط. لا أنبوب على قيمة ترجعها الدالة من داخل العملية.

## شرح المثال

- الواجهة لا تحمل `@IsEmail`. الزينة مكانها الصنف.
- استيراد القيمة يبقي الصنف في الملف المترجم.
- الإرجاع كائنًا يطابق `MemberView` يرضي TypeScript ولا يمر على `ValidationPipe`، لأن الأنبوب على الداخل.
- لو بدّلت المعامل إلى `MemberView`، الجسم كله يمر مهما كان `email`.

## ماذا يحدث داخليًا؟

`tsc` يحذف الواجهات. الصنف يبقى ويُستدعى `plainToInstance` عليه داخل الأنبوب حين يكون `metatype` ذلك الصنف. بيانات الزينة التي كتبتها `class-validator` تُقرأ من الصنف عبر `reflect-metadata`.

الاستيراد النوعّي لا يصدر ربطًا وقت التشغيل، فيصبح `metatype` بلا البناء الذي تتوقعه. الأعراض: كل طلب ينجح، أو السلوك يتغير حسب إعداد الاستيراد. العلاج استيراد قيمة.

الأنواع العامة مثل `CreateMemberDto[]` لا تصدر بيانات عناصرها. ذلك حد آخر ذُكر مع `ParseArrayPipe`، وهو قريب من حد الواجهة: ما لا يوجد وقت التشغيل لا يُفحص.

## مثال عملي

فريق وضع كل العقود في `member.types.ts` كواجهات لأن «DTO كلمة ثقيلة». `POST /members` يقبل `{ "email": "no" }`. بعد نقل الإنشاء إلى `CreateMemberDto` مع `@IsEmail()` و `ValidationPipe`، الطلب نفسه يرجع `400`. `MemberView` تبقى واجهة لنتيجة داخلية يجمعها المتحكم للاختبار. الحدّان مختلفان.

## أخطاء شائعة

- زينة على واجهة. TypeScript قد يرفضها أو تُهمل، والأنبوب لن يفحص.
- `import type` لصنف DTO في المتحكم.
- جعل كل الأشكال أصنافًا بما فيها ناتج حساب داخلي لا يعبر الأنبوب. الواجهة أخف هناك.
- توقع أن `implements` ينسخ الزينة من الواجهة. الزينة لا تعيش على الواجهة أصلًا.

## متى نستخدمه؟

الصنف على `@Body()` و `@Query()` و `@Param()` حين يكون المعامل كائن قواعد. الواجهة لشكل داخلي أو لناتج لا يفحصه `ValidationPipe`.

## متى لا نستخدمه؟

لا تضع واجهة مكان صنف الدخل. لا تستورد الصنف كنوع فقط. لا تبنِ صنفًا لكل متغير محلي.

## مقارنة

| | `interface` | `class` DTO |
| --- | --- | --- |
| بعد `tsc` | تختفي | تبقى |
| زينة `class-validator` | غير متاحة للأنبوب | تُقرأ وقت التشغيل |
| `ValidationPipe` | يتخطى الفحص | يفحص |
| استعمال مناسب | شكل داخلي | عقد يدخل الأنبوب |

## تمارين

### تمرين سهل

لماذا لا يكفي `interface` لـ `ValidationPipe`؟

<details>
<summary>الحل</summary>

لأنها تُمسح عند الترجمة ولا تترك بيانات زينة يقرأها الأنبوب.

</details>

### تمرين متوسط

ما الفرق بين `import { CreateMemberDto }` و `import type { CreateMemberDto }` في ملف المتحكم؟

<details>
<summary>الحل</summary>

الأول يبقي الصنف وقت التشغيل فيصله `metatype`. الثاني يُحذف، والأنبوب لا يرى البناء.

</details>

### تمرين متقدم

تريد شكل ناتج الخدمة `{ id, email }` بلا فحص أنبوب، وشكل الدخل مع `@IsEmail()`. أين الواجهة وأين الصنف؟

<details>
<summary>الحل</summary>

الدخل `class CreateMemberDto`. الناتج `interface MemberView` أو صنف خرج منفصل إن كان سيمر على تحويل لاحق. لا تجعل الواجهة معامل `@Body()`.

</details>

## أسئلة مراجعة

- ماذا يصدر TypeScript عن الواجهة؟
- أي استيراد يُمسح؟
- هل `implements` تنقل الزينة؟
- أين تبقى الواجهة مفيدة داخل الخدمة؟

## الخلاصة

عقد `ValidationPipe` صنف يُستورد كقيمة. الواجهة عقد مترجم لشكل لا يفحصه ذلك الأنبوب. خلطهما على `@Body()` يفتح الحد.

## ماذا بعد؟

الدرس التالي يفصل ما يراه المترجم عما يبقى في Node بعد تشغيل الملف.

## المصادر الرسمية

- [Validation](https://docs.nestjs.com/application/validation)
- [Pipes](https://docs.nestjs.com/pipes)
