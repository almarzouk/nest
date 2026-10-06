# Workspaces و Libraries

## الهدف من هذا الدرس

جمع واجهة رف وعامل التصدير في مساحة عمل واحدة، ومشاركة وحدة العضوية عبر مكتبة، مع Rspack كمُجمّع افتراضي للمساحة.

## المشكلة التي يحلها

نسخ `MembershipService` بين مستودعين فيفترقان. أو حزمة npm داخلية لكل تعديل صغير. أو علم `webpack` قديم في `nest-cli.json` بينما Nest 12 يتوقع `builder`.

## الفكرة ببساطة

الوضع القياسي تطبيق واحد ومترجم `tsc`. إضافة تطبيق ثانٍ بـ `nest generate app` تحوّل المجلد إلى monorepo: مجلد `apps` و `nest-cli.json` فيه `"monorepo": true` و `"builder": "rspack"`. المكتبة `nest g library` تعيش في `libs` ولا تملك `main.ts`. الاستيراد عبر بادئة مثل `@app/membership`. علما `webpack` و `webpackConfigPath` مسار قديم؛ الخيار الحالي `builder`.

## التعريف التقني

`nest generate app worker` ينقل `src` التطبيق الأصلي تحت `apps` ويضيف التطبيق الجديد. التحويل يفترض هيكل Nest المعتاد (`src` و `test`). `nest start` يشغّل المشروع الافتراضي من خاصية `"root"`. `nest start worker` يسمي المشروع. المكتبة: `type` هو `library` و `entryFile` هو `index`. البادئة الافتراضية `@app` ما لم تُضبط `defaultLibraryPrefix`. في ESM يشير `paths` في `tsconfig.json` إلى `./libs/membership/src/index.ts`. Rspack ينتج ملفًا واحدًا يجمع التطبيق ومكتباته. للملفات المنفصلة اضبطوا `builder` على `tsc` أو `swc`. خيارات المترجم على مستوى المشروع تستبدل مصفوفة `assets` العليا ولا تدمجها؛ `includeLibraryAssets` ينسخ أصول مكتبة عند بناء التطبيق.

## Mental Model

```text
apps/api        HTTP رف
apps/worker     تصدير بلا نسخ الكود
libs/membership وحدة العضوية
builder: rspack
```

## أبسط مثال

```bash
cd raf-api
nest generate app worker
nest g library membership
```

بعدها `nest start` يشغّل التطبيق الافتراضي، و `nest build worker` يبني العامل. المكتبة تُستورد ولا تُشغَّل وحدها.

## شرح المثال

- التحويل يعيد ترتيب المجلدات. نفّذوه على فرع نظيف.
- `@app/membership` اسم المسار لا اسم حزمة في `npm`.
- Rspack هنا قرار CLI للمساحة لا استبدال Express داخل التطبيق.

## ماذا يحدث داخليًا؟

1. المخطط يضع `"monorepo": true` ويكتب `projects` لكل تطبيق ومكتبة.
2. كل مشروع يأخذ `tsconfig.app.json` أو `tsconfig.lib.json` يمتد من جذر المساحة.
3. `tsc` لا يفهم تشغيل الاختبار. Vitest يحتاج `resolve.alias` يطابق `paths`. مشروع CommonJS على Jest يضع `moduleNameMapper` كما تذكر صفحة المكتبات.
4. `nest build api` يمر على المكتبات المستوردة ويجمعها لأن المُجمّع الافتراضي Rspack.
5. علم `compilerOptions.webpack: true` مهمل. استبدلوه بـ `"builder": "rspack"`.

## مثال عملي

```json
{
  "monorepo": true,
  "root": "apps/api",
  "compilerOptions": {
    "builder": "rspack",
    "deleteOutDir": true,
    "tsConfigPath": "apps/api/tsconfig.app.json"
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { MembershipModule } from '@app/membership';

@Module({ imports: [MembershipModule] })
export class AppModule {}
```

`organizationId` يبقى وسيط الدوال في المكتبة. المكتبة لا تقرأ جسم الطلب.

## أخطاء شائعة

- تحويل مشروع غيّر مكان `src` فيفشل المخطط أو يبعثر الملفات.
- الإبقاء على `--webpack` و `--webpackPath` في سكربتات مساحة Nest 12.
- نسيان alias الاختبار فيمر `nest build` ويفشل `vitest`.
- مكتبة فيها `listen` كأنها تطبيق.

## متى نستخدمه؟

عندما تتشارك تطبيقات الفريق الوحدات يوميًا وتريدون بناءً واحدًا للأصول وقواعد oxlint.

## متى لا نستخدمه؟

لمستودع رف وهو ما زال عملية HTTP واحدة. الوضع القياسي و `tsc` يكفي. حزمة npm تبقى للتوزيع خارج المستودع.

## مقارنة

| | الوضع القياسي | مساحة العمل |
| --- | --- | --- |
| المترجم الافتراضي | `tsc` | `rspack` |
| المكتبة | حزمة npm يدوية | `libs` و `nest g library` |
| `package.json` | لكل تطبيق | واحد للجذر |

## تمارين

### تمرين سهل

أي أمر يحوّل مشروعًا قياسيًا إلى monorepo؟

<details>
<summary>الحل</summary>

`nest generate app` لتطبيق ثانٍ. المكتبة وحدها تأتي بعد أن تكون المساحة موجودة، والأمر الموثق للتحويل في الصفحة هو إضافة تطبيق.

</details>

### تمرين متوسط

لماذا ملف Rspack واحد لا يعفيكم من alias في Vitest؟

<details>
<summary>الحل</summary>

`paths` يفهمها مترجم TypeScript والمُجمّع عند البناء. Vitest يحل الوحدات وقت الاختبار بقاعدته. ضعوا `resolve.alias` نحو `libs/membership/src`.

</details>

### تمرين متقدم

أصول المكتبة لا تظهر في `dist` التطبيق مع أن `assets` معرفة في جذر `nest-cli.json`. أين تُضاف؟

<details>
<summary>الحل</summary>

خيارات المشروع لا تُدمج مع العليا. ضعوا اسم المكتبة في `includeLibraryAssets` على التطبيق، أو أعيدوا تعريف `assets` على المشروع نفسه.

</details>

## أسئلة مراجعة

- أين تعيش المكتبة وما `entryFile`؟
- ما البادئة الافتراضية؟
- أي `builder` يكتبه Nest 12 للمساحة؟
- أي علم مترجم صار قديمًا؟

## الخلاصة

مساحة العمل تنسق تطبيقات رف ومكتبة العضوية في `nest-cli.json`. المُجمّع الافتراضي Rspack. علما webpack القديمان يُستبدلان بـ `builder`. المكتبة تُستورد ولا تُشغَّل وحدها.

## ماذا بعد؟

إن بقيتم على الوضع القياسي وأردتم ترجمة أسرع من `tsc` أثناء التطوير، الدرس التالي SWC.

## المصادر الرسمية

- NestJS — Workspaces: https://docs.nestjs.com/cli/monorepo
- NestJS — Libraries: https://docs.nestjs.com/cli/libraries
