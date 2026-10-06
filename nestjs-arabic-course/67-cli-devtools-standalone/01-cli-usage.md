# أوامر Nest CLI

## الهدف من هذا الدرس

إنشاء مشروع رف وتشغيله وتوليد ملفاته بأوامر `nest` كما يوثقها NestJS 12 في أكتوبر 2026.

## المشكلة التي يحلها

نسخ مجلد قديم يخلط نظام الوحدات وأداة الاختبار. أمر `generate` يدويًا ينسى تسجيل المتحكم في الوحدة. نسخة CLI عامة قديمة تولّد شكلًا لا يطابق الحزم في المشروع.

## الفكرة ببساطة

`@nestjs/cli` يبني الهيكل ويولّد الملفات ويشغّل البناء. `nest new` يسأل عن نظام الوحدات: ESM هو الافتراضي ويأتي مع Vitest، و CommonJS يأتي مع Jest. التطبيق المولَّد يستخدم oxlint و Prettier. المترجم الافتراضي في الوضع العادي `tsc`. أوامر مثل `build` و `start` و `generate` و `info` تكفي ليوم العمل قبل أي أداة أسرع.

## التعريف التقني

التثبيت الذي تفترضه الوثائق: `npm install -g @nestjs/cli`، أو `npx @nestjs/cli@latest` بلا تثبيت عام. ثنائي CLI يعمل على Node.js 20.11 أو أحدث، بينما مخططات `nest new` و `nest generate` و `nest upgrade` في `@nestjs/schematics` تطلب Node.js 22.22.3+ أو 24.15+ أو 26+. `nest new my-nest-project` ثم `npm run start:dev` يفتح http://localhost:3000 ويعيد الترجمة عند حفظ الملف. `nest generate controller projects` (والاسم المختصر `g`) ينشئ الملفات ويسجّلها. `nest info` يعرض نسخ الحزم. `nest upgrade` يرفع المشروع إلى الإصدار الرئيسي الأحدث. `nest --help` و `nest generate --help` يعرضان الخيارات. علم `--dry-run` أو `-d` يطبع ما سيُنشأ بلا كتابة.

## Mental Model

```text
nest new  → ESM + Vitest + oxlint + Prettier
nest g    → ملف + تسجيل في الوحدة
nest start --watch → ترجمة ثم عملية
```

## أبسط مثال

```bash
npx @nestjs/cli@latest new raf-api
cd raf-api
npm run start:dev
```

عند السؤال اختاروا ESM. سكربت `start:dev` في المشروع المولَّد هو مسار التطوير. لا تكتبوا ESLint في `package.json`؛ المولَّد يضع oxlint و Prettier.

## شرح المثال

- ESM يعني `"type": "module"` واستيرادًا بلاحقة `.js` في ملفات TypeScript.
- الاختبار `vitest run` لا `jest` ما دمتم على الافتراضي.
- CommonJS خيار صريح حين تحتاجونه، ومعه Jest. لا تخلطوا المشغّلين في مستودع واحد بلا قرار.

## ماذا يحدث داخليًا؟

1. `new` ينسخ الهيكل القياسي: `src` و `nest-cli.json` و `tsconfig.json` و `package.json`.
2. الوضع القياسي يترجم بـ `tsc`. وضع المساحة الأحادية، لاحقًا، يبدّل الافتراضي إلى Rspack.
3. `generate` يشغّل مخطط `@nestjs/schematics` ويحدّث الوحدة.
4. `start:dev` يراقب الملفات ويعيد البناء. هذا إعادة تشغيل للتطوير، ودرس إعادة التحميل الساخن يفصله عن HMR.
5. ثنائي Node بلا ICU يفشل في أوامر تعتمد التدويل. `node -p process.versions.icu` يجب أن يطبع نسخة لا `undefined`.

## مثال عملي

قبل توليد `ProjectsController` في رف:

```bash
npx nest generate controller projects --dry-run
npx nest generate controller projects
npx nest info
```

`--dry-run` يعرض المسارات. بعد التوليد تأكدوا أن `ProjectsModule` يستورد المتحكم. `nest build` يملأ `dist` للنشر، و `node dist/main.js` هو شكل التشغيل بعد البناء في درس Docker من المشروع الختامي.

## أخطاء شائعة

- CLI عام قديم ومشروع Nest 12 فيختلف شكل `nest-cli.json`.
- اختيار CommonJS ثم استيراد ESM بلاحقة `.js` و Vitest معًا.
- إضافة ESLint لأن مقالًا قديمًا ذكره، بينما المولَّد على oxlint.
- تشغيل `generate` من مجلد غير جذر المشروع فيُنشأ الملف خارج `src`.

## متى نستخدمه؟

عند إنشاء الخدمة، وتوليد وحدة أو متحكم أو مزود، ومعرفة النسخ المثبتة، وترقية الإطار.

## متى لا نستخدمه؟

لا يغني عن قراءة `nest-cli.json` حين يصبح المستودع مساحة عمل. ولا يستبدل `@nestjs/resilience` أو صندوق الصادر.

## مقارنة

| الأمر | الناتج في رف |
| --- | --- |
| `nest new` | تطبيق ESM و Vitest و oxlint و Prettier |
| `nest g resource projects` | وحدة ومسار REST للمسودات |
| `nest build` | `dist` للحاوية |
| `nest info` | نسخ `@nestjs/*` على الجهاز |

## تمارين

### تمرين سهل

ما مشغّل الاختبار في مشروع `nest new` إن تركتم ESM؟

<details>
<summary>الحل</summary>

Vitest. CommonJS هو الذي يأتي مع Jest. الاثنان يستخدمان oxlint.

</details>

### تمرين متوسط

لماذا `npx @nestjs/cli@latest` أحيانًا أوضح من تثبيت `-g`؟

<details>
<summary>الحل</summary>

التثبيت العام نسخة واحدة لكل المشاريع على الجهاز. `npx` يشغّل نسخة تديرونها مع المشروع. الوثائق تفترض العام لسهولة الأمثلة وتذكر `npx` بديلًا.

</details>

### تمرين متقدم

`node -p process.versions.icu` طبع `undefined` وفشل `nest new`. ماذا ينقص الثنائي؟

<details>
<summary>الحل</summary>

دعم ICU. ثنائيات Node الرسمية من موقع Node تتضمنه. استبدلوا الثنائي ثم أعيدوا الأمر.

</details>

## أسئلة مراجعة

- ما الافتراضي: ESM أم CommonJS؟
- أي أداة lint في التطبيق المولَّد؟
- ماذا يفعل `--dry-run`؟
- أي مترجم يفترضه الوضع القياسي؟

## الخلاصة

`nest` ينشئ رف على ESM و Vitest و oxlint و Prettier، ويولّد الملفات، ويبني بـ `tsc` في الوضع العادي. نسخة CLI ومخططات التوليد تتبعان حدود Node المذكورة في الصفحة.

## ماذا بعد؟

حين تتشارك واجهة رف وعامل التصدير الكود، يتحول المجلد إلى مساحة عمل ومكتبات.

## المصادر الرسمية

- NestJS — CLI overview: https://docs.nestjs.com/cli/overview
