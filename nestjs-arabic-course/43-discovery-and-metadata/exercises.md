# تمارين الفصل

## سهل

ماذا تضعه أولًا في مصفوفة `getAllAndOverride` حتى تغلب الدالة الصنف؟

<details>
<summary>الحل</summary>

`context.getHandler()` ثم `context.getClass()`. أول قيمة معرّفة تفوز.

</details>

## متوسط

`@FeatureFlag('experimental')` على خدمة غير مذكورة في أي `providers`. هل يراها `getProviders({ metadataKey })`؟

<details>
<summary>الحل</summary>

لا. الاكتشاف يعدد المسجّل في الرسم. الملف الذي لم يُسجل لا يدخل الحاوية ولو لصق المزخرف.

</details>

## متقدم

حارس واحد على HTTP و `@MessagePattern`. اكتب من أين يأتي معرف الفاعل في كل قناة، وماذا لا تستقبل `BillingService`.

<details>
<summary>الحل</summary>

من `switchToHttp().getRequest().user.id` عندما `getType()` هو `http`، ومن `switchToRpc().getData().userId` عندما يكون `rpc`. الخدمة تستقبل `userId: string` ولا تستقبل `ExecutionContext` ولا `Request`.

</details>
