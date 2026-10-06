# ماذا بعد الاعتمادية؟

رف تستطيع أن تحد النداء الخارجي، وتخزن نتيجة `POST` بمفتاح، وتكتب رسالة النشر في معاملة Prisma عبر `@nestjs/outbox`، وتنتظر أيامًا في `@nestjs/workflows`، وتشغّل التصدير مرة بـ `@OnOneInstance` على مخزن مشترك. إعادة المحاولة بلا مفتاح، والقفل في الذاكرة، و `setTimeout` لأسبوع، لا تغطي هذه الحالات.

ما يشغّل هذا يوميًا أوامر CLI: مساحات، SWC، REPL، وإعادة تحميل، وأدوات مطور، وتطبيق بلا HTTP، وأوامر `nest-commander`.

[CLI و Devtools و Standalone](../67-cli-devtools-standalone/README.md)
