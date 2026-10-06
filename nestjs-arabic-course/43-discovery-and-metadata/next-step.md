# التمهيد للفصل التالي

`Reflector` قرأ مفتاحًا، و `DiscoveryService` عدد المسجّلين، و `ExecutionContext` سمى القناة. الرمز الذي حقنته حتى الآن غالبًا صنف في `providers: [BooksService]`. هذا اختصار لـ `{ provide: BooksService, useClass: BooksService }`.

الفصل التالي يفتح الشكل الطويل: `useClass` يبدل التنفيذ، و `useValue` يثبت كائنًا أو مزيف اختبار، و `useFactory` يبني القيمة من اعتماديات، و `useExisting` اسم ثانٍ للنسخة نفسها. ثم الرموز النصية و `Symbol` لأن الواجهة تُمحى، ثم المصنع الذي يرجع `Promise` وهو ما لمسته في المزودات غير المتزامنة.

جملة العبور: metadata تصف الدالة، والمزود المخصص يقرر أي قيمة تقف خلف الرمز عند الحقن.

[Custom Providers المتقدمة](../44-advanced-custom-providers/README.md)
