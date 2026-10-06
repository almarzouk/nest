# تمارين الفصل

## تمرين سهل — الأداة

اربطوا: ESM، مساحة جديدة، تطوير أسرع على تطبيق قياسي، جلسة طرفية، مهمة بلا HTTP.

1. Vitest
2. `"builder": "rspack"`
3. `npm run start -- -b swc`
4. `repl()`
5. `createApplicationContext`

<details>
<summary>الحل</summary>

1 افتراضي `nest new`. 2 monorepo في Nest 12. 3 باني SWC عبر سكربت npm. 4 ملف `repl.ts`. 5 تطبيق مستقل بلا `listen`.

</details>

## تمرين متوسط — أعلام قديمة

سكربت المساحة فيه `nest build --webpack --webpackPath webpack-hmr.config.js`. ماذا تستبدلون؟

<details>
<summary>الحل</summary>

علما webpack مهملان. مساحة v12 تبني بـ Rspack عبر `"builder": "rspack"`. المراقبة اليومية تبقى `nest start --watch`. وصفة HMR في الوثائق تستهدف webpack و CommonJS.

</details>

## تمرين متقدم — أمر التصدير

`ExportInvoicesCommand` نجح على آلة المطوّر وصمت في Cron، والمنظمة لم تُفحص. ماذا ينقص التشغيل والتصميم؟

<details>
<summary>الحل</summary>

`CommandFactory.run` يطفئ المسجّل بلا وسيط ثانٍ مثل `['error']`. المعرف علم `@Option` يُتحقق منه في المهمة مقابل سجل المنصة. `CommandFactory` يغلق السياق، لكن `@OnOneInstance` ما زال لازمًا إن شغّلت عدة آلات. الحارس لا يعمل لأن الأمر ليس مسار HTTP.

</details>
