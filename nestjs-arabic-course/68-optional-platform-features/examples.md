# أمثلة عملية

ثلاث إضافات على رف بعد ما يعمل المسار الأساسي.

## المثال 1 — لغة وبريد

`I18nModule.forRoot` بـ `ar` و `en` و `I18nService.t` من خدمة singleton. تأكيد النشر صنف `Mailable` و `FileMailTransport` في التطوير. الإرسال من معالج الصندوق بعد الالتزام لا من مسار المتصفح. الاختبار يقرأ `InMemoryMailTransport`.

## المثال 2 — غلاف ودفق

`uploadToDisk` على قرص `covers` يرفض غير PNG/JPEG من البايتات. المفتاح يُحفظ مع `organizationId` الموثوق. الفاتورة على قرص خاص. الواجهة تفتح `@Sse('projects/:id/activity')` حتى `{ cover: 'ready' }` ثم يغلق `finalize` المؤقت.

## المثال 3 — عملية أم دالة أم هجين

واجهة رف تبقى `listen` في حاوية. عامل بلا مسارات قد يكون `createApplicationContext` داخل دالة بعد قياس البداية الباردة، لا نقلًا لجدول webpack في الوثائق. إن بقيت الرسائل في نفس العملية: `connectMicroservice` ثم `init` ثم `startAllMicroservices`، وبلا `inheritAppConfig` حتى يقرأ المعالج المنظمة من الحمولة.
