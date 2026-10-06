# التمهيد للفصل التالي

السجل يشرح الفشل بعد وقوعه. الاختبار يثبت السلوك قبل النشر: خدمة بمعزل، وحدة مجمّعة، وطلب HTTP كامل.

الفصل `38-testing` يفرّق الأنواع الثلاثة. قالب `nest new` الحديث على ESM يستعمل Vitest، وقالب CommonJS يستعمل Jest. `TestingModule` و `overrideProvider` و Supertest على `app.getHttpServer()` بعد `init` بلا `listen` حيث يكفي ذلك. قاعدة الاختبار ليست قاعدة الإنتاج.

افتح [`38-testing`](../38-testing/README.md) عندما تستطيع الإشارة إلى سطر واحد فيه `bookId` و `requestId` بلا سر.
