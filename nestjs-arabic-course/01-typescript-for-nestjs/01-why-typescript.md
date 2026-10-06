# لماذا TypeScript مع NestJS

## الهدف من هذا الدرس

معرفة ماذا يضيف TypeScript فوق JavaScript الذي تعرفه، ولماذا Nest تُكتب وتُوثَّق به.

## المشكلة التي يحلها

JavaScript ينتظر التشغيل ليكتشف أن `book.price` ليست رقمًا، أو أنك مررت خدمة غلط إلى دالة. في مشروع فيه عشرات الملفات، هذا الاكتشاف يأتي من طلب حقيقي أو من اختبار متأخر.

Nest تقرأ أنواع معاملات الـ constructor لتعرف ماذا تحقن. من غير معلومات نوع، الحاوية لا تعرف أن المعامل `UsersService` وليس أي كائن آخر. TypeScript هو المكان الذي تُكتب فيه هذه المعلومات.

التوثيق يقول: Nest مكتوبة بـ TypeScript وتدعم JavaScript العادي، لكن JavaScript يحتاج Babel لأن الإطار يستخدم ميزات لغة حديثة. مشاريع CLI الجديدة تُولَّد بـ TypeScript ووضع `strict` مفعّل.

## الفكرة ببساطة

TypeScript هو JavaScript مع أنواع تُفحص قبل التشغيل. المترجم يحوّل الملف إلى JavaScript. الأنواع نفسها تُمسح من الناتج، إلا ما يُطلب إصداره كـ Metadata للـ Decorators.

## التعريف التقني

TypeScript لغة من Microsoft تُترجم إلى JavaScript. نظام الأنواع فيها هيكلي (structural): الشكل المهم، لا اسم الصنف فقط. `strict` في قوالب Nest يفعّل فحوصات مثل `strictNullChecks` حتى لا تعامل `null` كأنه قيمة صالحة بصمت.

## Mental Model

```text
أنت تكتب book.ts
      |
      |  tsc أو SWC يفحص الأنواع
      v
أخطاء قبل التشغيل
      |
      v
JavaScript الناتج يشغّله Node
      |
      v
Nest تقرأ Metadata الأنواع إن فُعّل emitDecoratorMetadata
```

## أبسط مثال

```ts
function priceLabel(price: number): string {
  return `${price} USD`;
}

priceLabel(40);
```

`priceLabel('40')` يفشل أثناء الفحص لأن `'40'` نص لا عدد. وقت التشغيل، بعد الترجمة، الدالة مجرد JavaScript. الفحص حدث قبل أن يصل الطلب.

## شرح المثال

- `price: number` عقد للمعامل.
- `: string` عقد لقيمة الإرجاع.
- القالب النصي يعمل في JavaScript أيضًا. الجديد هو رفض النداء الخاطئ قبل Node.

## ماذا يحدث داخليًا؟

المترجم يبني جدول رموز ويفحص التوافق. ثم يحذف التعليقات النوعية وينتج JavaScript. لذلك نوع على جسم HTTP لا يحمي الإنتاج، كما رأيت في فصل JSON: القيمة تأتي من `JSON.parse` بعد الترجمة.

الاستثناء المهم لـ Nest: مع `emitDecoratorMetadata` يكتب المترجم استدعاءات `Reflect.metadata` تحمل أنواع التصميم، مثل أنواع معاملات الـ constructor. هذا استثناء محدود، لا إحياء لكل الأنواع. الدرس 15 يفصله. Nest 12 ما زالت تعتمد هذا المسار مع Decorators التقليدية (`experimentalDecorators`)، لا على Decorators المرحلة 3 من معيار JavaScript وحدها، لأن Metadata التصميم مربوطة بمسار المترجم الحالي.

## مثال عملي

دالة إنشاء كتاب تمنع تمرير عنوان رقمي وأنت تكتب الكود:

```ts
interface NewBook {
  title: string;
  price: number;
}

function draft(input: NewBook): string {
  return input.title;
}
```

`draft({ title: 'Refactoring', price: 40 })` مقبول. `draft({ title: 5, price: 40 })` مرفوض قبل التشغيل. الطلب القادم من الشبكة ما زال يحتاج Validation لاحقًا، لأن المترجم لا يرى JSON العميل.

## أخطاء شائعة

- الظن أن TypeScript يبطئ الطلبات لأنه "يفحص الأنواع أثناء الخدمة". الفحص قبل التشغيل. الناتج JavaScript.
- إطفاء `strict` في أول يوم. القالب يفعله لسبب. التوثيق يقول يمكنك ضبط `"strict": false` إن أردت الخروج، والمسار الأفضل إبقاؤه.
- كتابة `as any` لتمرير كل فحص. تلغي سبب الأداة.

## متى نستخدمه؟

في كل ملف Nest في هذه الدورة. الأمثلة الرسمية TypeScript، والحقن يعتمد على الأنواع.

## متى لا نستخدمه؟

سكربت عقدة من عشرين سطرًا خارج التطبيق، إن لم يشارك أنواع المشروع. داخل Nest لا يوجد سبب لخلط ملفات بلا أنواع.

## مقارنة

| | JavaScript | TypeScript |
| --- | --- | --- |
| أين يُكتشف الخطأ الشكلي | عند تشغيل ذلك السطر | عند الفحص والترجمة |
| ناتج التشغيل | هو نفسه | JavaScript يترجمه المترجم |
| دور Nest | ممكن مع Babel | المسار الافتراضي للـ CLI |

## تمارين

### تمرين سهل

لماذا `function total(price: number)` لا تمنع عميل HTTP من إرسال `"price": "40"`؟

<details>
<summary>الحل</summary>

لأن النوع يُفحص على الكود الذي تترجمه، لا على بايتات الطلب. JSON يصبح قيمة JavaScript بعد التشغيل، والمترجم انتهى قبلها.

</details>

### تمرين متوسط

اذكر شيئًا يبقى بعد الترجمة وشيئًا يُحذف.

<details>
<summary>الحل</summary>

يبقى منطق الدالة وقيمها. يُحذف تعليق النوع مثل `: number`، إلا Metadata التصميم التي تُصدر عمدًا بجانب Decorators عند تفعيل الخيار.

</details>

### تمرين متقدم

لماذا تبقي Nest على Decorators التقليدية في الإصدار 12 بدل الانتقال الفوري إلى Decorators المعيارية وحدها؟

<details>
<summary>الحل</summary>

لأن حقن الاعتماديات وما شابهه يقرأ Metadata أنواع التصميم من المترجم. هذا المسار مرتبط بالـ Decorators الحالية وبـ `emitDecoratorMetadata`. الانتقال قبل بديل رسمي مستقر يكسر الاستدلال على أنواع الـ constructor. التوثيق والمجتمع ما زالا على هذا المسار في Nest 12.

</details>

## أسئلة مراجعة

- ماذا يفحص TypeScript، ومتى؟
- لماذا القالب يفعّل `strict`؟
- أي جزء من الأنواع تحتاجه الحاوية بعد الترجمة؟

## الخلاصة

TypeScript عقد على الكود قبل التشغيل، وهو لغة Nest الافتراضية. الأنواع العادية لا تعبر الشبكة، وMetadata التصميم استثناء نبني عليه الحقن لاحقًا.

## ماذا بعد؟

نكتب أنواعًا بسيطة ونرى أين يخمنها المترجم وحده: Types و Inference.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — Introduction: https://docs.nestjs.com/
- TypeScript — Handbook, The Basics: https://www.typescriptlang.org/docs/handbook/2/basic-types.html
- TypeScript — `emitDecoratorMetadata`: https://www.typescriptlang.org/tsconfig/emitDecoratorMetadata.html
