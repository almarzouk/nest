# التمهيد للفصل التالي

الأخطاء الآن أصناف ورموز، والفلتر يكتب الجسم من غير أن يطوّل كل دالة. ما زال المتحكم يقرأ `request.user` و `request.headers` في التوقيع، والحارس يبحث عن مفتاح نصي مثل `'roles'` إن ألصقته بـ `SetMetadata` يدويًا في كل ملف.

الفصل `16-custom-decorators` يبني زينة تخفي هذه القراءة: `createParamDecorator` لمعامل مثل المستخدم الحالي، و `SetMetadata` للبيانات، و `Reflector` لقراءتها. الشكل الذي يولّده Nest CLI 12 للزينة هو `Reflector.createDecorator()`. ثم `@CurrentUser` و `@Roles` و `@Public`. الأخيرة هي العلامة التي تفتح الصحة وتسجيل الدخول أمام حارس عام من غير `exclude` في Middleware.

الزينة القديمة في TypeScript تحتاج `experimentalDecorators` و `emitDecoratorMetadata` و `reflect-metadata`. مشاريع CLI تفعّلها لأن الحقن يعتمد على بيانات المعاملات.

افتح [`16-custom-decorators`](../16-custom-decorators/README.md) عندما تستطيع أن تشير إلى `request.user` في متحكم وتقول: هذا مكان زينة معامل لا مكان فلتر.
