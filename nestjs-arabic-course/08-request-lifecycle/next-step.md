# التمهيد للفصل التالي

الخريطة الآن ثابتة. أول ما يلمس الطلب بعد دخوله هو Middleware: `app.use` ثم ربط الوحدات، و `@Global()` قبل الجذر ثم الأبعد. Guards لا تبدأ إلا بعد `next()`، واستثناء هذه المرحلة لا يراه إلا Exception Filter العام.

الفصل `09-middleware` يبني هذه المحطة: الدالة البسيطة، والصنف الذي يتبع `NestMiddleware`، و `configure` مع `MiddlewareConsumer`، و `forRoutes` و `exclude`، وترتيب التنفيذ عندما تتعدد الروابط. ستستخدم الخريطة التي هنا حتى لا يتحول Middleware إلى Guard أو Pipe.

افتح [`09-middleware`](../09-middleware/README.md) وأنت قادر على وضع `requestId` في موضعه من القائمة قبل أي حارس.
