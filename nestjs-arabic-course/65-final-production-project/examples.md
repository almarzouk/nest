# أمثلة عملية

مسار دار النيل كما تتراكم الشرائح، كل مثال يضيف طبقة لا يعيد المشروع كله.

## المثال 1 — من الحساب إلى المسودة

تسجيل سارة يعيد 201 بلا تجزئة. `POST /organizations` ينشئ `org_nile` بعضوية `owner` من الإثبات. `POST /projects` بترويسة `org_nile` وجسم `{ "title": "موسم الهجرة" }` يعيد 201 و `draft`. `GET /projects` لمكتبة بلا صفوف يعيد 200 و `data: []`.

## المثال 2 — نشر بلا تفعيل كاذب

الصف `trial` يسمح بـ `POST /projects/:id/publish`. المعاملة تكتب `published` وتدقيق `project.publish` وصف ويب هوك `pending`. المهمة `project.cover` تُجدول بعد الالتزام. `GET /billing/success` يقرأ `trial` ولا يكتب `active`.

## المثال 3 — جارة واختبار

مكتبة `org_other` تطلب معرّف الموسم بترويستها فتأخذ 404. وثيقة OpenAPI لا تحتوي `organizationId` في جسم الإنشاء. صورة التشغيل `node dist/main.js`. اختبار Vitest يثبّت الحالتين: 201 ثم 404، و 200 مع `data: []`.
