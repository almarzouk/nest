# تمارين الفصل

## سهل

ما النطاق إن لم تكتب `scope` في `@Injectable()`؟

<details>
<summary>الحل</summary>

`Scope.DEFAULT`. نسخة واحدة للتطبيق، وخطافات دورة الحياة تعمل عليها.

</details>

## متوسط

`AuditService` حقنت رمز `REQUEST` لتقرأ عنوان IP، و `OrdersController` يحقن `AuditService`. ماذا يصبح عمر المتحكم، وما البديل الذي يبقي الاثنين Singleton؟

<details>
<summary>الحل</summary>

المتحكم يصعد إلى `REQUEST` لأن العدوى لا تُلغى. اقرأ IP في Guard أو Interceptor ومرره وسيطًا، أو ضعه في `AsyncLocalStorage`، وأبقِ `AuditService` على الافتراضي من غير حقن `REQUEST`.

</details>

## متقدم

عشرة مستأجرين، لكل منهم اتصال قاعدة خاص، والكتالوج كله يحقن مصدر البيانات. لماذا `REQUEST` العادي مكلف هنا، وما شرطا `durable` قبل أن تجربه؟

<details>
<summary>الحل</summary>

كل طلب يعيد بناء المتحكم والكتالوج. مع عشرة مفاتيح يمكن استراتيجية `ContextIdFactory` تجمع `x-tenant-id`، والمصدر يُصرَّح `Scope.REQUEST` مع `durable: true` حتى تُعاد الشجرة للمستأجر نفسه. العدد الكبير من المفاتيح يجعل الخريطة تسريبًا، وعندها البقاء على عميل مشترك مع `where` أصح إن كان المخطط يسمح.

</details>

## سؤال ربط مع دورة الحياة

لماذا `$connect` على `PrismaService` العابرة أو نطاق الطلب لا يغني عن خطاف الإقلاع؟

<details>
<summary>الحل</summary>

خطافات التطبيق لا تُستدعى على `REQUEST`. والعابر يبني نسخة لكل مستهلك لا مرة للتطبيق، فتفتح أكثر من عميل. مكان الاتصال `onModuleInit` على Singleton واحد.

</details>
