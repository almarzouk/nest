# تمارين الفصل

أغلق الدروس ثم أجب. الحلول تحت `الحل`.

## سهل

رتّب: `listen`، `onApplicationBootstrap`، `onModuleInit`، `onApplicationShutdown`، `onModuleDestroy`، `beforeApplicationShutdown`.

<details>
<summary>الحل</summary>

`onModuleInit`، `onApplicationBootstrap`، `listen`، ثم عند الإنهاء: `onModuleDestroy`، `beforeApplicationShutdown`، إغلاق الاتصالات، `onApplicationShutdown`.

</details>

## متوسط

`PrismaService` على `Scope.REQUEST` وفيه `onModuleInit` ينادي `$connect`. ماذا يحدث لخطاف التطبيق، وما الكلفة التي ستدرسها في الفصل التالي؟

<details>
<summary>الحل</summary>

خطافات دورة حياة التطبيق لا تُستدعى على أصناف الطلب. كل طلب سيبني شجرة جديدة إن بقي النطاق `REQUEST`، وهذا أغلى من Singleton يفتح المجمع مرة. الإبقاء على الافتراضي مع `$connect` مرة في `onModuleInit` هو المسار المناسب للعميل المشترك.

</details>

## متقدم

Kubernetes يرسل `SIGTERM` وينتظر 30 ثانية ثم `SIGKILL`. اكتب أين تضع: `enableShutdownHooks`، `$disconnect`، وإيقاف مؤقت الفواتير. ماذا لا يفعله `app.close()` وحده للعملية؟

<details>
<summary>الحل</summary>

`enableShutdownHooks()` في `bootstrap` حتى تدخل الإشارة. `$disconnect` في `onModuleDestroy` لخدمة Prisma. `clearInterval` في `onModuleDestroy` أيضًا حتى لا يبقى مقبض. `app.close()` يشغّل السلسلة ويغلق السيرفر ولا يستدعي `process.exit`. إن بقي مؤقت، العملية تظل حتى `SIGKILL`.

</details>

## سؤال ربط

لماذا تفتح قاعدة البيانات في وحدة ورقة لا في `AppController`؟

<details>
<summary>الحل</summary>

لأن الإقلاع يمشي من الأوراق إلى الجذر وينتظر كل وحدة. ورقة القاعدة تُنهى قبل المتحكمات التي ستطلبها، والإيقاف يعكس الاتجاه فيغلق الجذر جدولة العمل قبل قطع الورقة.

</details>

## جولة ثانية

### سهل — من يستقبل الإشارة؟

`onModuleDestroy` أم `beforeApplicationShutdown`؟

<details>
<summary>الحل</summary>

`beforeApplicationShutdown(signal)` و `onApplicationShutdown(signal)`. `onModuleDestroy` بلا وسيط إشارة.

</details>

### سهل — متى تعمل الخطافات بلا enableShutdownHooks؟

<details>
<summary>الحل</summary>

عند `app.close()` الصريح، كما في الاختبار. الإشارات تحتاج `enableShutdownHooks()`.

</details>

### متوسط — connect على الناقل

أين `ClientProxy.connect`، وأين `close`؟

<details>
<summary>الحل</summary>

`connect` في `onApplicationBootstrap` حتى تكتمل الوحدات قبل الحجز. `close` في `onModuleDestroy` قبل خطوات قطع HTTP الأخيرة.

</details>

### متوسط — Windows

الحاوية على Linux تُغلق بـ `SIGTERM`. التطوير على Windows بـ Ctrl+C. ماذا يصل في كل بيئة؟

<details>
<summary>الحل</summary>

`SIGTERM` يصل على Linux بعد التفعيل. على Windows يصل `SIGINT` ولا يصل `SIGTERM`. السلسلة نفسها تعمل إن وصلت الإشارة المدعومة.

</details>

### متقدم — ترتيب الإغلاق

الجذر يجدول فاتورة كل دقيقة. الورقة Prisma. اكتب ترتيب `onModuleDestroy` ولماذا لا تقطع القاعدة أولًا.

<details>
<summary>الحل</summary>

الإيقاف من الجذر إلى الأوراق. الجذر يمسح `setInterval` أولًا. الورقة تنادي `$disconnect` بعد أن تتوقف الوحدات الأعلى عن الاستعلام. القطع في الجذر قبل ذلك يترك كتابات على عميل مغلق.

</details>

### متقدم — process.exit

بعد `app.close()` بقيت العملية. ما الذي تفحصه، وما الذي لا تستدعيه داخل الخطاف؟

<details>
<summary>الحل</summary>

افحص المؤقتات والمقابض. `app.close()` لا ينهي العملية وحده. لا تستدعِ `process.exit` داخل الخطاف لإخفاء مقبض. أزل المقبض ثم دع الإغلاق يتم.

</details>
