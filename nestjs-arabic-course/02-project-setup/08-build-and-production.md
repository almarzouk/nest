# البناء والإنتاج

## الهدف من هذا الدرس

التمييز بين كود TypeScript الذي تكتبه وناتج JavaScript الذي تشغّله Node.

## المشكلة التي يحلها

Node لا يشغّل زينة TypeScript ومصدر `.ts` كبيئة إنتاج من غير خطوة بناء تضبطها Nest. تشغيل `ts-node` على الخادم العام أسلوب تطوير.

## الفكرة ببساطة

`npm run build` يكتب JavaScript إلى `dist/`. `npm run start:prod` يشغّل ذلك الناتج.

## التعريف التقني

CLI يستدعي المترجم حسب `nest-cli.json` و `tsconfig`. الناتج يعكس ESM أو CommonJS الذي اخترته. في monorepo الإصدار 12 يجعل Rspack البنّاء الافتراضي، وأعلام webpack القديمة deprecated لصالح `--builder rspack`. تطبيقنا المفرد في أول الدورة لا يحتاج monorepo.

## Mental Model

```text
src/**/*.ts  ->  build  ->  dist/**/*.js  ->  node
```

## أبسط مثال

```bash
npm run build
npm run start:prod
```

## شرح المثال

البناء يفشل إن وُجد خطأ نوع في `strict`. أصلح المصدر ثم أعد البناء. `start:prod` لا يراقب الملفات. هذا ما تريده في بيئة مستقرة.

## ماذا يحدث داخليًا؟

المترجم يمسح الأنواع ويبقي Metadata الزينة إن كانت الخيارات مفعلة في `tsconfig` الذي يقرأه البناء فعلًا، لا في ملف آخر نسيته. `nest build` يستخدم مسار `tsconfig` المشار إليه من إعداد CLI. ناتج ESM ما زال يحمل استيرادات `.js`.

## مثال عملي

بعد أول بناء افتح `dist/main.js` ولاحظ اختفاء واجهة لو كتبت واحدة، وبقاء صنف الخدمة. لا تعدّل `dist` يدويًا. أضف `dist` إلى التجاهل في Git إن لم يكن القالب قد فعل ذلك.

## أخطاء شائعة

- تشغيل `node src/main.ts` وتوقع نجاح الزينة دائمًا.
- نسخ المشروع إلى الخادم من غير `build` ومن غير `node_modules` إنتاجية.
- تفعيل webpack قديم لأن درسًا ذكره. في monorepo الحالي المسار الموثّق هو Rspack.

## متى نستخدمه؟

قبل أي تجربة إنتاج أو Docker لاحقًا.

## متى لا نستخدمه؟

لا تبنِ بعد كل سطر وأنت في `start:dev`. البناء خطوة تسليم.

## مقارنة

| | تطوير | إنتاج |
| --- | --- | --- |
| الأمر | `start:dev` | `build` ثم `start:prod` |
| الملفات | `src` | `dist` |
| المراقبة | نعم | لا |

## تمارين

### تمرين سهل

أي مجلد يكتب فيه البناء عادة؟

<details><summary>الحل</summary>

`dist`.

</details>

### تمرين متوسط

لماذا قد ينجح المحرر ويفشل `npm run build`؟

<details><summary>الحل</summary>

المحرر قد يستخدم `tsconfig` مختلفًا عن الذي يمر به `nest build`، أو لا يفحص ملفًا خارج المشروع المفتوح.

</details>

### تمرين متقدم

ما الذي يجب أن يبقى مفعّلًا في tsconfig البناء حتى يعمل الحقن بعد الترجمة؟

<details><summary>الحل</summary>

`experimentalDecorators` و `emitDecoratorMetadata`، مع تحميل `reflect-metadata` عند الإقلاع.

</details>

## أسئلة مراجعة

- ماذا يشغّل `start:prod`؟
- لماذا لا نعدّل `dist`؟
- أي بنّاء أصبح افتراضي monorepo في CLI 12؟

## الخلاصة

التطوير يراقب `src`. الإنتاج يشغّل JavaScript مبنيًا في `dist` مع Metadata الزينة سليمة.

## ماذا بعد؟

القالب يجهّز lint وتنسيقًا حتى لا نؤجلهما إلى ما لا نهاية.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — CLI overview: https://docs.nestjs.com/cli/overview
- NestJS — CLI scripts: https://docs.nestjs.com/cli/scripts
- NestJS — Deployment: https://docs.nestjs.com/deployment
