# ماذا بعد Stripe؟

فوترة رف خلف وحدة: Checkout يعيد رابطًا، الاشتراك صف محلي يُزامَن من حدث، و `constructEvent` على الجسم الخام، ومفتاح إنشاء الجلسة غير قيد `event.id`. المتصفح على success لا يكتب `active`.

عندما ينشر الكتالوج كتابًا، خدمات أخرى داخل رف تريد أن تعرف من غير أن تستورد `BooksService` كلها. ذلك فرق بين حدث داخل النموذج وحدث يعبر الحدود، واتساق قد يتأخر.

[Event-Driven Architecture](../63-event-driven-architecture/README.md)
