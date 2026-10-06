# ESM و CommonJS

## الهدف من هذا الدرس

قراءة استيرادات قالب Nest 12 وعدم كسرها بأسلوب `require` من درس قديم.

## المشكلة التي يحلها

نظاما الوحدات في Node يصفان كيف يصدّر ملف رمزًا وكيف يحمّله ملف آخر. خلطهما في المشروع نفسه ينتج أخطاء تحميل غامضة.

## الفكرة ببساطة

ESM يستخدم `import` و `export` وهو افتراضي `nest new`. CommonJS يستخدم `require` و `module.exports` ويبقى خيارًا في CLI مع Jest.

## التعريف التقني

في ESM مسار الاستيراد النسبي يحدد الملف، ولذلك تكتب `.js` حتى داخل TypeScript لأن المسار يصف الناتج. الحزم الرسمية لـ Nest 12 ESM. مشروع CommonJS ما زال قادرًا على تحميلها على نسخ Node المدعومة عبر `require(esm)`. الترقية إلى 12 لا تهاجر مشروعك إلى ESM تلقائيًا؛ `nest upgrade` يترك ذلك قرارًا لك.

## Mental Model

```text
ESM:  import { AppModule } from './app.module.js'
CJS:  const { AppModule } = require('./app.module')
```

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

export class Placeholder {}
```

## شرح المثال

السطر الأول يستورد رمزًا مُسمّى من حزمة. السطر الثاني يستورد من ملف محلي مع الامتداد الذي سينتج بعد البناء. `export` يجعل الرمز متاحًا للمستوردين. لا يوجد `require` في هذا القالب.

## ماذا يحدث داخليًا؟

Node يحمّل ESM بشكل ثابت: الاستيرادات في أعلى الملف وتُحل قبل تنفيذ الجسم. CommonJS يحمّل `require` حين يصل التنفيذ إليه. أدوات مثل SWC و Vitest في القالب مضبوطة على النظام الذي اخترته. تغيير حقل type في package.json من غير تحويل الاستيرادات يكسر التشغيل.

## مثال عملي

أبقِ مشروع الدورة ESM. عندما تنسخ مثالًا من مقال يستخدم `require`، حوّله إلى `import` قبل لصقه. الحزم التي لا توفّر ESM تحتاج قرارًا لاحقًا، لا خلطًا عشوائيًا في `main.ts`.

## أخطاء شائعة

- حذف `.js` من الاستيراد المحلي لأن الملف على القرص `.ts`. في قالب ESM البناء يفشل أو المحمّل لا يجد الملف.
- استخدام `import type` لخدمة تُحقن.
- افتراض أن كل مثال على الإنترنت CommonJS ما زال هو افتراضي Nest.

## متى نستخدمه؟

ESM لكل مشروع جديد في هذه الدورة.

## متى لا نستخدمه؟

لا تحوّل مكتبة قديمة كبيرة إلى ESM في منتصف فصل تعليمي. إن وُجد مشروع CommonJS قائم، `nest upgrade` يبقيه ويبقي Jest.

## مقارنة

| | ESM | CommonJS |
| --- | --- | --- |
| الصيغة | import/export | require/module.exports |
| اختبار القالب | Vitest | Jest |
| امتداد الاستيراد المحلي | `.js` في المصدر TS | غالبًا بلا امتداد في الأسلوب القديم |

## تمارين

### تمرين سهل

ما صيغة الاستيراد في القالب الافتراضي؟

<details><summary>الحل</summary>

`import { Symbol } from '...'`.

</details>

### تمرين متوسط

لماذا الامتداد `.js` داخل ملف TypeScript؟

<details><summary>الحل</summary>

لأنه مسار ناتج ESM بعد الترجمة، و TypeScript يطابقه مع المصدر أثناء الفحص.

</details>

### تمرين متقدم

مشروع CommonJS على Node 22 يحمّل `@nestjs/core` بنجاح. ما الميزة التي جعلت ذلك ممكنًا من غير تحويل المشروع كله؟

<details><summary>الحل</summary>

دعم `require(esm)` في نسخة Node التي يشترطها Nest 12.

</details>

## أسئلة مراجعة

- ما افتراضي `nest new`؟
- ماذا يحدث لو حذفت `.js` من استيراد محلي في ESM؟
- هل الترقية إلى 12 تجبر ESM؟

## الخلاصة

القالب الحديث ESM. اكتب `import` واترك امتداد `.js` في المسارات المحلية.

## ماذا بعد؟

نقرأ الملفات الخمسة داخل `src/`.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — CLI overview: https://docs.nestjs.com/cli/overview
- NestJS — CLI scripts: https://docs.nestjs.com/cli/scripts
