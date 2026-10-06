# nest new

## الهدف من هذا الدرس

إنشاء مشروع TypeScript تشغّله وترى Hello World.

## المشكلة التي يحلها

جمع `@nestjs/core` و `reflect-metadata` و `tsconfig` يدويًا ممكن، والتوثيق يذكر الحد الأدنى من الحزم، لكنه طريق أطول للدرس الأول.

## الفكرة ببساطة

`nest new bookstore` ينشئ مجلدًا، يسأل عن نظام الوحدات، يثبّت الحزم، ويولد `src/`.

## التعريف التقني

من First steps: CLI يسأل ESM (الافتراضي، مع Vitest) أو CommonJS (مع Jest). يسأل أيضًا عن NestJS Observe. في طرفية تفاعلية الافتراضي نعم. في CI يُتخطى السؤال ولا تُضاف Observe إلا مع `--observe`. `--no-observe` يلغيها. القالب يفعّل `strict`. بديل من غير CLI: استنساخ `typescript-starter` ثم `npm install`، أو كتابة الملفات مع الحزم `@nestjs/core` و `@nestjs/common` و `rxjs` و `reflect-metadata`.

## Mental Model

```text
nest new bookstore
  -> سؤال ESM أو CommonJS
  -> سؤال Observe
  -> package.json و src/
```

## أبسط مثال

```bash
nest new bookstore
cd bookstore
```

## شرح المثال

الاسم يصبح مجلد المشروع وحقل `name`. ادخل المجلد قبل أوامر `npm run`. إن أردت تخطي الأسئلة في سكربت: مرر خيارات المدير ونظام الوحدات كما يعرضها `nest new --help` على نسختك، ولا تحفظ أعلام مقال قديم من غير مراجعة `--help`.

## ماذا يحدث داخليًا؟

المولّد ينسخ قالبًا ويثبت الاعتماديات بمدير الحزم الذي تختاره في السؤال. ESM يعني type module في package.json غالبًا وامتدادات .js في استيرادات TypeScript لأن الناتج ESM. CommonJS يبقي require ممكنًا والأسلوب الأقدم في الاختبارات مع Jest.

## مثال عملي

أنشئ `bookstore` بـ ESM و `strict` كما هو الافتراضي، ومن غير Observe في أول أسبوع حتى تبقى ملفات الإقلاع هي ما يشرحه الفصل التالي. أعد تفعيل المراقبة عندما تصل إلى فصل Observability.

## أخطاء شائعة

- اختيار CommonJS فقط لأن درسًا قديمًا يستخدم `require`. الدورة تتبع ESM الافتراضي.
- تعديل عشر حزم قبل أول تشغيل.
- إنشاء المشروع داخل مجلد الدورة التعليمية نفسها وخلط ملفات الشرح مع `src`. اجعله مجلدًا شقيقًا للتجارب.

## متى نستخدمه؟

لكل تطبيق في هذه الدورة ما لم نبنِ مثالًا من ملفين لشرح الإقلاع.

## متى لا نستخدمه؟

لا تستخدم القالب الجاهز ثم تحذف `AppService` قبل أن ترى الربط مرة واحدة.

## مقارنة

| طريق | الناتج |
| --- | --- |
| `nest new` | قالب كامل وأمر `nest` |
| git clone للـ starter | قريب منه من غير سؤال CLI |
| تثبيت الحزم يدويًا | أنت تكتب `main.ts` |

## تمارين

### تمرين سهل

ما الأمر الذي ينشئ مجلد المشروع؟

<details><summary>الحل</summary>

`nest new bookstore` مع استبدال الاسم.

</details>

### تمرين متوسط

ما فارق الاختبار بين اختيار ESM و CommonJS في السؤال الأول؟

<details><summary>الحل</summary>

ESM يجهّز Vitest. CommonJS يجهّز Jest.

</details>

### تمرين متقدم

لماذا تظهر الاستيرادات كـ `'./app.module.js'` داخل ملف `.ts`؟

<details><summary>الحل</summary>

في ESM يشير المسار إلى ملف الناتج بعد الترجمة. TypeScript يحل `.js` إلى المصدر `.ts` أثناء التطوير. هذا أسلوب قوالب Nest الحديثة لا خطأ إملاء.

</details>

## أسئلة مراجعة

- ما نظام الوحدات الافتراضي؟
- ماذا يفعل `--no-observe`؟
- ما الحزم الدنيا إن رفضت القالب؟

## الخلاصة

`nest new` يعطيك تطبيق ESM صارمًا يشغّل Hello World. اقبله قبل أن تخصصه.

## ماذا بعد؟

نفهم ESM لأن الاستيراد في القالب يفاجئ من اعتاد CommonJS.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — CLI overview: https://docs.nestjs.com/cli/overview
- NestJS — CLI scripts: https://docs.nestjs.com/cli/scripts
