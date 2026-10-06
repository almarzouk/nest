# أمثلة عملية

نشر مشروع في رف ثم تصدير ليلي، كل أداة في مكانها.

## المثال 1 — زر الإنشاء

`POST /projects` عليه `@Idempotent({ required: true })` ونطاق المستخدم. الإعادة لا تنشئ مسودة ثانية. `@Retry` من `@nestjs/resilience` ليس على هذا المسار.

## المثال 2 — الرسالة مع الصف

`prisma.$transaction(async (tx) => { update; outbox.add(..., tx) })`. لا مصفوفة استعلامات. `@OnOutboxMessage('project.published')` يحمّل المنظمة من الحمولة. تذكير بعد سبعة أيام تدفق `@nestjs/workflows` لا `setTimeout`.

## المثال 3 — الثالثة فجرًا

`@Cron` مع `@OnOneInstance()` ومخزن قفل في Postgres لا في الذاكرة. اختبار ثلاث نسخ يعد تصديرًا واحدًا. اقتباس الناقل البطيء `@Timeout('2s')` فيرد 504 ولا يغيّر `published`.
