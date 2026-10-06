# ماذا بعد الإصدارات؟

رف تستطيع تشغيل عقدين في عملية واحدة: URI بعد البادئة العامة، أو ترويسة تسميها `header`، أو `Accept` مع `key: 'v='`. الصفحة التي تصف ذلك هي Versioning، على https://docs.nestjs.com/techniques/versioning ومسار HTTP الحالي https://docs.nestjs.com/http/versioning. طلب بلا نسخة مطابقة يعيد 404 ما لم تضع `defaultVersion` أو `VERSION_NEUTRAL`.

الآلية لا تقول إن كان `price` كائنًا أو عددًا، ولا إن كانت القائمة الفارغة 200. هذا عقد تصميم قبل اختيار الديكور.

الفصل التالي يكتب العقد: أسماء الموارد، الرموز، Idempotency، والصفحة كاتفاق، وشكل الخطأ، ثم مكان الإصدار داخل هذا الاتفاق.

[تصميم REST API](../56-rest-api-design/README.md)
