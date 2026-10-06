# التمهيد للفصل التالي

حذف الدورة أعاد الاستيراد إلى سهم نحو ورقة. تلك الورقة في الأمثلة كانت `@Module` ثابتًا: المزودات مكتوبة في الزخرفة. الخطوة التالية أن الورقة نفسها تحتاج خيارًا من المستورد، مثل سابقة معرّف أو عنوان قاعدة، من غير أن تنسخ الوحدة لكل ميزة.

ذلك Dynamic Module: دالة ثابتة `forRoot` أو `register` ترجع كائنًا فيه `module` والمزودات المبنية من الخيار. `forRootAsync` و `registerAsync` تنتظران `ConfigService`. من يعيد التصدير يصدّر الصنف ولا يستدعي `forRoot` مرة ثانية.

ستمر أيضًا على `ConfigurableModuleBuilder` و `ModuleRef` الذي لمحنا إليه لفك طرف من دورة، وعلى التحميل الكسول والمزودات غير المتزامنة. جملة العبور: الدورة تُحذف برسم شجرة، والشجرة التي تحتاج خيارًا خارجيًا تُكوَّن عند الاستيراد لا بنسخ الملفات.

## حد تعبره

الورقة الثابتة لا تقبل عنوان قاعدة من المستورد. نسخ الملف لكل ميزة يعيد الدورة بشكل ملفات. الدالة الثابتة ترجع `DynamicModule` وفيه `module` إلزامي. `forRoot` و `forRootAsync` مرة في الجذر، وغالبًا `global: true`. `register` و `registerAsync` لكل مستهلك، ومن غير `global` إن كان الرمز النصي مشتركًا بين نداءين.

من يعيد التصدير يكتب `exports: [DatabaseModule]` بعد أن استُدعي `forRoot` مرة. نداء ثانٍ يعيد بناء المزودات.

`registerAsync` أحد ثلاثة فقط: `useFactory` أو `useClass` أو `useExisting`. `useClass` يطبّق `create()` إلا إن غيّرت اسم الدالة. `ConfigurableModuleBuilder` يولّد `ConfigurableModuleClass` و `MODULE_OPTIONS_TOKEN` و `setClassMethodName('forRoot')` و `setExtras` حتى لا يدخل `isGlobal` في رمز الخيارات.

`ModuleRef.get` للـ Singleton و `strict` افتراضيًا. `resolve` للنطاق. `create` لصنف غير مسجل. لا تستدعِ `get` في الـ `constructor`.

`LazyModuleLoader.load` بعد `import()` يخبئ الوحدة ويرجع `ModuleRef`. خطافات تلك الوحدة لا تُستدعى. لا تحمّل متحكمًا أو بوابة أو resolver أو middleware أو وحدة عامة بعد `listen`.

مصنع `useFactory` الذي يرجع `Promise` يُنتظر قبل بناء من يعتمد عليه. هذا غير `forRootAsync` الذي يرجع خيارات الوحدة. الكائن العادي بلا خطاف إيقاف.

## تمرين عبور

سابقتان لمعرف المستخدم ومعرف الطلب. أي أسلوب، وهل الوحدة عامة؟

<details>
<summary>الحل</summary>

`register` أو `registerAsync` داخل كل ميزة، بلا `global: true`، حتى لا يختلط الرمز. `forRoot` مرة واحدة يبقى لعنوان القاعدة المشترك.

</details>

[Dynamic Modules](../42-dynamic-modules/README.md)
