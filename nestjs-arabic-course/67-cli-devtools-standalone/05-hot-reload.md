# Hot Reload

## الهدف من هذا الدرس

تمييز إعادة تشغيل `nest start --watch` التي يستخدمها رف عن استبدال الوحدات الساخن، ومعرفة أن وصفة HMR الحالية تستهدف webpack المهمل.

## المشكلة التي يحلها

كل حفظ يعيد إقلاع Nest وثواني انتظار. نسخ سكربت `nest build --webpack --webpackPath ...` من مقال قديم إلى مساحة v12 يعيد علمًا مهملًا. أو توقع أن HMR ينسخ ملفات `.graphql` و HTML القوالب وحده.

## الفكرة ببساطة

`npm run start:dev` في التطبيق المولَّد يترجم من جديد ويعيد تشغيل العملية. هذا مسار رف اليومي. التحميل الساخن يبقي العملية ويستبدل الوحدات. صفحة Hot reload تبني ذلك على webpack و `module.hot`، وتنص أن باني webpack صار مهملًا في v12 وأن Rspack هو المُجمّع الافتراضي للمساحات، وتنصح المشاريع الجديدة بـ `--builder rspack`. علم `manualRestart` يتيح كتابة `rs` لإعادة التشغيل يدويًا أثناء المراقبة.

## التعريف التقني

المراقبة من CLI: `nest start --watch` أو سكربت `start:dev`. في `nest-cli.json` يمكن `"manualRestart": true` تحت `compilerOptions` فيصبح الاختصار `rs` متاحًا. وصفة HMR تثبّت `webpack-node-externals` و `run-script-webpack-plugin` و `webpack`، وتضيف `HotModuleReplacementPlugin`، وفي `main.ts` تنتظر `module.hot.data.closePromise` قبل `listen` حتى لا يظهر `EADDRINUSE`، مع `forceCloseConnections: !!module.hot`. الصفحة تقول إن الأمثلة CommonJS لهذا السبب (`bootstrap()` بلا `await` في المستوى الأعلى). سكربت الوصفة `nest build --webpack --webpackPath webpack-hmr.config.js --watch` يستخدم الأعلام المهملة. مشروع جديد على ESM لا يُنقل إلى ذلك المسار. أصول `compilerOptions.assets` مع `watchAssets` هي طريق CLI لنسخ JSON و HTML عند البناء والمراقبة على `tsc` و `swc`.

## Mental Model

```text
حفظ → watch → إيقاف العملية → إقلاع
HMR (وصفة webpack) → استبدال وحدات داخل العملية
مشروع جديد → راقبوا أو Rspack، لا --webpack
```

## أبسط مثال

```bash
npm run start:dev
```

```json
{
  "compilerOptions": {
    "manualRestart": true,
    "assets": ["mail/templates/**/*", "i18n/**/*.json"],
    "watchAssets": true
  }
}
```

غيّروا عنوان كتاب في الذاكرة أو في كود المتحكم. العملية تعيد الإقلاع والطلب التالي يرى الكود الجديد. `rs` ثم Enter يعيد التشغيل بلا حفظ إن فُعّل العلم.

## شرح المثال

- الإقلاع الكامل يعيد تشغيل `onModuleInit` واتصال Prisma. هذا واضح في السجلات.
- القوالب وأدلة الترجمة ملفات ليست TypeScript. `watchAssets` ينسخها إلى `dist` عند الحفظ. الصفحة تحذّر أن webpack لا ينسخ الأصول تلقائيًا ولا يتوافق مع مسارات glob الثابتة لكيانات TypeORM.
- Rspack في المساحة يختصر وقت البناء إلى ملف واحد. ذلك ليس سطر HMR في الوصفة.

## ماذا يحدث داخليًا؟

1. وضع المراقبة يعيد ترجمة الملف المتغير ثم يعيد تشغيل Node.
2. المنفذ يُغلق مع العملية فيتحرر قبل العملية التالية.
3. في وصفة webpack، `dispose` يخزن وعد `app.close()` لأن الإضافة لا تنتظر الـ callback. العملية التالية تنتظر الوعد ثم تنادي `listen`.
4. `forceCloseConnections` يغلق الاتصالات حتى لا يبقى المنفذ محجوزًا.
5. علم `--webpack` في مساحة v12 يشير إلى المسار المهمل. `"builder": "rspack"` هو ما يكتبه CLI للمساحة.

## مثال عملي

مطوّر رف على ESM يبقي:

```json
"start:dev": "nest start --watch"
```

ويضع قوالب البريد في `assets`. لا يضيف `webpack-hmr.config.js` ليرى الحفظ أسرع. إن صار البناء هو البطء على الوضع القياسي فالباني `swc` من الدرس السابق. إن صارت المساحة كبيرة فالمُجمّع `rspack`. قياس الوقت على مشروعكم يسبق نقل الوصفة القديمة.

## أخطاء شائعة

- لصق `--webpack` و `--webpackPath` في monorepo جديد.
- توقع بقاء جلسة REPL أو ذاكرة الطلب بعد إعادة تشغيل watch.
- قالب HTML لا يظهر لأن `assets` غائبة و webpack لا ينسخها.
- `module.hot` في `main.ts` الخاص بـ ESM بينما التشغيل `nest start --watch` بلا تلك الوصفة.

## متى نستخدمه؟

`start:dev` كل يوم. `manualRestart` حين تريدون إعادة إقلاع بلا تعديل ملف. قياس Rspack أو SWC حين تكون الترجمة هي الانتظار.

## متى لا نستخدمه؟

لا تعتمدوا وصفة webpack HMR لمشروع رف الجديد. ولا تستبدلوا إعادة التحميل باختبار يثبت السلوك.

## مقارنة

| الأسلوب | ماذا يبقى حيًا | في Nest 12 |
| --- | --- | --- |
| `nest start --watch` | لا شيء من العملية السابقة | مسار التطبيق المولَّد |
| HMR عبر webpack | العملية مع استبدال الوحدات | الوصفة، والباني مهمل |
| Rspack | ناتج بناء مجمّع | افتراضي المساحة |

## تمارين

### تمرين سهل

ماذا يفعل `npm run start:dev` في مشروع `nest new`؟

<details>
<summary>الحل</summary>

يترجم ويعيد تحميل التطبيق عند تغيير المصدر، كما تصف نظرة CLI. هذا إعادة تشغيل تطوير لا وصفة `module.hot`.

</details>

### تمرين متوسط

لماذا تنتظر الوصفة `closePromise` قبل `listen`؟

<details>
<summary>الحل</summary>

`app.close()` غير متزامن و webpack لا ينتظر `dispose`. بلا الانتظار تحاول العملية الجديدة المنفذ ويظهر `EADDRINUSE`. `forceCloseConnections` يرافق ذلك.

</details>

### تمرين متقدم

مساحة رف على Rspack. أي سطر ترفضونه في `package.json`؟

<details>
<summary>الحل</summary>

`nest build --webpack --webpackPath webpack-hmr.config.js`. الأعلام مهملة. الإعداد `"builder": "rspack"` هو مسار المساحة في v12.

</details>

## أسئلة مراجعة

- ما فرق watch عن HMR؟
- أي باني تفضله الصفحة للمشاريع الجديدة؟
- ماذا يفعل `manualRestart`؟
- من ينسخ قوالب البريد إلى `dist`؟

## الخلاصة

تطوير رف اليومي هو `nest start --watch` مع `assets` عند الحاجة. التحميل الساخن الموثق مبني على webpack والصفحة تعتبر ذلك الباني مهملًا أمام Rspack.

## ماذا بعد؟

لرؤية الوحدات والمسارات وهي تعمل، الدرس التالي Nest Devtools.

## المصادر الرسمية

- NestJS — Hot reload: https://docs.nestjs.com/recipes/hot-reload
- NestJS — Workspaces: https://docs.nestjs.com/cli/monorepo
