# تمارين الفصل

أغلق الدروس ثم أجب. الحلول تحت `الحل`.

## سهل

أي مسار يُعاد تشغيل الحاوية إن فشل، وأيهما يُسحب من التوزيع فقط؟

<details>
<summary>الحل</summary>

فشل `/health/live` يعيد التشغيل. فشل `/health/ready` يسحب المرور ويبقي العملية.

</details>

## متوسط

اكتب سطر المؤشر لـ Prisma بمهلة ثانية، والمفتاح `database`.

<details>
<summary>الحل</summary>

`this.prismaHealth.pingCheck('database', this.prisma).withTimeout(1000)` داخل `health.check` وعلى دالة عليها `@HealthCheck()`.

</details>

## متوسط — الكاش

Redis يخزّن أغلفة الكتب فقط. الصفحات تُقرأ من Postgres عند غيابه. أي حالة تُرجعها، وما رمز HTTP؟

<details>
<summary>الحل</summary>

`indicator.degraded('cache unreachable, serving without cache')`. الحالة العامة `degraded` و HTTP `200` ما دام مؤشر القاعدة `up`. النتيجة تحت `info` لا `error`.

</details>

## متقدم

طابور الفواتير على Redis، والجاهزية تستعمل `degraded` عند فشل `PING`. الطلبات تُقبل ولا تُنشر. ماذا تغيّر في المؤشر وفي مهلة المسبار؟

<details>
<summary>الحل</summary>

اجعل المؤشر `attempt(() => this.redis.ping()).withTimeout(1000)` أو `down` حتى يصير `status` القيمة `error` و HTTP `503`. اجعل مهلة المسبار أطول قليلًا من مهلة المؤشر حتى يصل الرد قبل أن ييأس المنسّق.

</details>

## متقدم — الإغلاق

`gracefulShutdownTimeoutMs: 5000` وخطافات الإيقاف مطفأة. Kubernetes يرسل `SIGTERM`. ماذا يرى مسبار الجاهزية؟

<details>
<summary>الحل</summary>

Terminus لا يستقبل الإشارة بلا `enableShutdownHooks()`، فلا يظهر `shutting_down`. فعّل الخطافات في `bootstrap`. التأخير نفسه ينطبق على `SIGTERM` لا على `SIGINT`.

</details>

## سؤال ربط

لماذا `checkHeap` على الحياة أعاد تشغيل نسخًا سليمة بعد جمع القمامة؟

<details>
<summary>الحل</summary>

`checkHeap` يقارن `heapUsed` بعتبة بالبايت. الذروة ليست دليلًا أن الحلقة ماتت. الحياة تبقى ردًا رخيصًا. فحص الذاكرة، إن لزم، على مسار تشخيص لا على مسبار يقتل الحاوية.
</details>
