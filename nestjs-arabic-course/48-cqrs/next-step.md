# التمهيد للفصل التالي

CQRS الذي أبقيناه يعيش داخل عملية واحدة: `CommandBus` و `QueryBus` ينتظران، و `EventBus` يخبر معالجات في نفس الذاكرة. عندما يجب أن يُنشر الإلغاء في عملية أخرى لها إصدار مستقل وحد فشل مستقل، تغادر الحافلة المحلية إلى ناقل.

الفصل التالي يبدأ من الـ Monolith الذي أنت فيه، ثم الـ Modular monolith الذي بنيته بالوحدات، ثم متى تصبح الخدمة الصغيرة عملية ثانية. بعدها أساليب الطلب والحدث، وحياد المنصة، ثم `@MessagePattern` و `@EventPattern` و `ClientProxy`، ثم النواقل TCP و Redis و RabbitMQ و Kafka و NATS و gRPC و MQTT. `RpcException` لخطأ يعبر الناقل لا كـ `HttpException` فقط.

جملة العبور: حدث CQRS ليس رسالة شبكة. رسالة الشبكة لها ناقل ونمط وحد فشل.

[Microservices](../49-microservices/README.md)
