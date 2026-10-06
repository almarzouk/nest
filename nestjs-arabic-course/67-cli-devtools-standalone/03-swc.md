# SWC

## الهدف من هذا الدرس

تسريع حلقة تطوير رف بباني `swc` عبر `npm run start -- -b swc`، مع إبقاء فحص الأنواع على `tsc`.

## المشكلة التي يحلها

`tsc` على كل حفظ يبطئ التجربة. إغلاق فحص الأنواع كليًا يمرر `organizationId` خاطئًا إلى الإنتاج. أو إعداد webpack قديم لمساحة تستخدم Rspack أصلًا.

## الفكرة ببساطة

SWC مترجم Rust. الوثائق تقدّر أنه أسرع بنحو عشرين مرة من `tsc` في الترجمة، ونظرة CLI تقول حوالي عشر مرات في بناء التطوير. هو لا يفحص الأنواع. `--type-check` يشغّل `tsc --noEmit` بجانب SWC. في مساحة Nest 12 المُجمّع الافتراضي Rspack ويترجم عبر `swc-loader` داخله؛ قسم webpack في صفحة SWC يخص الباني المهمل فقط. سكربت npm يحتاج `--` حتى تصل الأعلام إلى `nest start`.

## التعريف التقني

`npm i --save-dev @swc/cli @swc/core`. التشغيل:

```bash
npm run start -- -b swc
```

لأن سكربت `start` هو `nest start`، و `--` يمرر `-b` إلى CLI. الشكل المباشر في الوثائق: `nest start -b swc` أو `nest start --builder swc`. المراقبة: `nest start -b swc -w`. في `nest-cli.json`: `"compilerOptions": { "builder": "swc", "typeCheck": true }`. SWC يخرج صيغة الوحدات التي يخرجها `tsc`: ESM حين `"type": "module"`، وإلا CommonJS. ملف `.swcrc` يقدر أن يغيّر ذلك؛ اتركوا الافتراضي لرف. إضافات CLI مثل Swagger تُنفَّذ مع `--type-check` وتكتب ملف بيانات وصفية. الاستيراد الدائري مع TypeORM يُكسر بنوع `Relation<T>` حتى لا يُحفظ النوع في بيانات الانعكاس.

## Mental Model

```text
حفظ الملف → SWC يترجم
typeCheck → tsc --noEmit بجانب الترجمة
monorepo → Rspack (swc-loader) لا -b swc فوق webpack
```

## أبسط مثال

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

```bash
npm run start -- -b swc --watch
```

العلم على سطر الأمر يغطي ما في الملف لتلك العملية. `typeCheck` يبقى من الملف إن لم تمرروا `--type-check`.

## شرح المثال

- `-b` اختصار `--builder`.
- بلا `typeCheck` قد يُشغَّل كود فيه خطأ نوع حتى يفشل وقت التشغيل.
- Vite/Vitest في مشروع ESM يستطيع أن يبني الاختبارات بـ `unplugin-swc` كما في قسم Vitest من الصفحة. ذلك إعداد اختبار لا بديل عن باني التطبيق.

## ماذا يحدث داخليًا؟

1. CLI يستدعي SWC على ملفات `src` ويكتب `dist` بنفس شكل الوحدات.
2. مع `typeCheck` تنطلق عملية `tsc` بلا إصدار ملفات، والأخطاء تظهر في الطرفية بينما الخادم يعمل على ناتج SWC.
3. الإضافات تولّد `metadata.ts` عند فحص الأنواع حتى تعمل الزخارف التي تقرأ تعليقات التصميم.
4. في monorepo، Rspack يحمّل `swc-loader` مسبقًا. لا تثبّتوا `swc-loader` مع `webpack.config.js` إلا إن بقيتم على الباني المهمل عن قصد.
5. استيراد دائري يحفظ نوع الخاصية في `design:type` فيفشل SWC. `Relation` أو `WrapperType` المحلي يقطع تلك الدورة.

## مثال عملي

رف ESM. غلاف المشروع يُنشأ في مهمة. إن ظهرت دورة بين `ProjectsService` و `CoversService` فالنوع على الخاصية المحقونة:

```ts
import { Inject, Injectable, forwardRef } from '@nestjs/common';
import type { CoversService } from './covers.service.js';

type WrapperType<T> = T;

@Injectable()
export class ProjectsService {
  constructor(
    @Inject(forwardRef(() => CoversService))
    private readonly covers: WrapperType<CoversService>,
  ) {}
}
```

هذا نمط الصفحة للدورات. الأفضل تفكيك الدورة بواجهة أصغر حين يسمح التصميم.

## أخطاء شائعة

- `npm run start -b swc` بلا `--` فيبتلع npm العلم.
- اعتبار SWC بديلًا لـ `tsc` في CI بلا `typeCheck` أو خطوة `tsc --noEmit`.
- نسخ إعداد webpack من صفحة SWC إلى مساحة Rspack.
- `"module"` داخل `.swcrc` يخالف `"type": "module"` فيُكسر الاستيراد.

## متى نستخدمه؟

على الوضع القياسي حين تصبح إعادة `tsc` هي انتظاركم اليومي، مع فحص أنواع مفعّل.

## متى لا نستخدمه؟

لا تستبدلوا به Rspack في monorepo إلا بقرار `builder` صريح تريدون معه ملفاتًا منفصلة (`tsc` أو `swc`). ولا تلغوا فحص الأنواع قبل الدمج.

## مقارنة

| المسار | الترجمة | فحص الأنواع |
| --- | --- | --- |
| قياسي `tsc` | أبطأ | داخل المترجم |
| `-b swc` | أسرع | مع `--type-check` فقط |
| monorepo Rspack | SWC عبر المُجمّع | ما زلتم تحتاجون `tsc` في CI |

## تمارين

### تمرين سهل

اكتبوا الأمر الذي يمرر الباني عبر سكربت `npm run start`.

<details>
<summary>الحل</summary>

`npm run start -- -b swc`

</details>

### تمرين متوسط

لماذا يمر بناء SWC ثم يفشل Prisma أو نوع المعرّف لاحقًا؟

<details>
<summary>الحل</summary>

SWC لا يفحص الأنواع. فعّلوا `typeCheck` أو شغلوا `tsc --noEmit` في CI.

</details>

### تمرين متقدم

المساحة على Rspack. هل تثبّتون `swc-loader`؟

<details>
<summary>الحل</summary>

لا لهذا المسار. الصفحة تقول إن monorepo منذ الإصدار 12 يترجم بـ SWC داخل Rspack. `swc-loader` مع webpack يخص الباني المهمل.

</details>

## أسئلة مراجعة

- ما حزم التطوير اللازمة؟
- أين يُكتب `builder`؟
- ماذا يخرج SWC لمشروع `"type": "module"`؟
- متى يُستخدم `Relation` أو `WrapperType`؟

## الخلاصة

`npm run start -- -b swc` يترجم رف بسرعة. الأنواع تُفحص بأمر مستقل. مساحة العمل تبقى على Rspack ما لم تختاروا `builder` آخر.

## ماذا بعد؟

لاستدعاء `ProjectsService` من الطرفية بلا طلب HTTP، الدرس التالي REPL.

## المصادر الرسمية

- NestJS — SWC: https://docs.nestjs.com/recipes/swc
- NestJS — CLI overview: https://docs.nestjs.com/cli/overview
