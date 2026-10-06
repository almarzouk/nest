# أمثلة عملية

طلب نشر واحد في رف كما يظهر للمشغّل.

## المثال 1 — ثلاث إشارات

المقياس `publish_duration_ms` يرتفع. السجل `{ msg, organizationId, bookId, requestId, traceId }` بمستوى log لا error. الأثر span اسمه `BooksService.publish` وتحته SQL. 404 على كتاب آخر لا يزيد عداد 500.

## المثال 2 — إقلاع

مشروع بلا لوحة: `nest new raf-api --no-observe` ثم OpenTelemetry إن لزم المجمع. مشروع يريد الأسماء الرسمية: `nest new raf-api --observe` ثم `instrument: ObserveInstrument` في `create`. Fastify يضع المحول ثانيًا والخيار ثالثًا. `/health` يبقى لـ Terminus.

## المثال 3 — دعم

الموظف يرسل `X-Request-Id` من الرد. البحث في السجل يجد النشر ورفض التوقيع لاحقًا لنفس المعرّف. `tenantId` في السطر يثبت المكتبة ولا يُؤخذ من جسم عُرض في الرسالة.
