# تمارين الفصل

أجب من غير فتح الدروس، ثم افتح الحل.

## سهل

1. من يختار `RegionalTax` عندما تطلب `DraftService` الصنف المجرد `TaxPolicy`؟
2. متى تُنشأ نسخة Singleton بالنسبة إلى `listen`؟

<details>
<summary>الحل</summary>

1. التسجيل في Module، مثل `{ provide: TaxPolicy, useClass: RegionalTax }`. الخدمة لا تنادي `new`.
2. أثناء حل الرسم في `NestFactory.create`، قبل قبول الاتصالات.

</details>

## متوسط

الإقلاع يقول إن الاعتمادية عند الفهرس 0 في `DraftService` لا تُحل. المُنشئ `(private readonly tax: TaxPolicy)`. اذكر ثلاثة أسباب مختلفة وأين تظهر في خطوات الحل.

<details>
<summary>الحل</summary>

1. `import type { TaxPolicy }` أو كون `TaxPolicy` واجهة: لا مفتاح في `design:paramtypes`.
2. الصنف مفتاح صحيح لكنه غير موجود في `providers` ولا وصل عبر `imports` و `exports`.
3. دورة تجعل الحل لا يجد ورقة يبدأ منها، والرسالة تذكر الرموز المتبقية.

`@Optional()` ليس إصلاحًا لسياسة إلزامية؛ هو يمرر `undefined`. `NEST_DEBUG=true` يطبع مسار الحل أثناء التشخيص.

</details>

## متقدم

`SequenceService` افتراضية فيها `next()` يزيد حقلًا. `ReportsService` و `DraftsController` يحقنانها من Module صدّرتها مرة واحدة. طلبان متتاليان على `next`، ثم اختبار ثانٍ ينشئ تطبيقًا جديدًا. ماذا يرى كل طرف؟ وأين يوضع معرف العضو الحالي بدل الحقل؟

<details>
<summary>الحل</summary>

الطلبان يريان التزايد لأن الخانة واحدة في الحاوية. Controller والتقرير يأخذان المرجع نفسه. الاختبار الثاني `NestFactory.create` جديد فيبدأ العدّاد من الصفر. معرف العضو وسيط دالة أو مزود `REQUEST`، لا حقل Singleton. `await` بين قراءة الحقل وكتابته يسمح لطلب آخر بالمرور على النسخة رغم أن الخيط واحد.

</details>
