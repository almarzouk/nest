# التمهيد للفصل التالي

صار عندك وصف وحدة يُبنى عند الاستيراد، ورمز خيار تحقنه الخدمة، وطريقة تجلب مزودًا بعد الإقلاع بـ `ModuleRef`. ما زال الإطار يعرف المزودات لأنك كتبتها في `providers`. الخطوة التالية أن تسأل الرسم نفسه: من يحمل metadata معينة، وأي دوال عليها، وما سياق التنفيذ الذي يميز HTTP عن RPC.

ذلك Discovery و Reflector و `ExecutionContext`. ستقرأ ما كتبه `@SetMetadata` والديكورات، وتصفّي المزودات بـ `DiscoveryService` بعد استيراد `DiscoveryModule`، وتفرّع الكود بـ `switchToHttp` و `switchToRpc` و `switchToWs` من غير أن تستورد أنواع Express في مكتبة مشتركة.

جملة العبور: الوحدة الديناميكية تُسجّل ما تعرفه عند الإقلاع، والاكتشاف يقرأ ما لصقته المزخرفات على الأصناف الحية.

## ما الذي تقرأه من الرسم

`ExecutionContext.getType()` يعيد `http` أو `rpc` أو `ws`. `getHandler` و `getClass` غذاء `Reflector`. `switchToHttp` و `switchToRpc` و `switchToWs` يفتحان القناة. لا تخزن `request` على Singleton.

`Reflector.get` يقرأ مفتاحًا. `getAllAndOverride` أول قيمة معرّفة تفوز، فضع المعالج قبل الصنف. `getAllAndMerge` يجمع. `SetMetadata` أو `Reflector.createDecorator`. الواجهة TypeScript تُمحى ولا تصلح رمز حقن.

`DiscoveryService` يحتاج `DiscoveryModule`. `getProviders` و `getControllers` مع `include` أو `metadataKey`. `DiscoveryService.createDecorator` ثم `getMetadataByDecorator`. الناتج `InstanceWrapper`. امسح عند الإقلاع لا في كل طلب.

## سيناريو

علم `@Public()` على دالة تسجيل الدخول وعلى صنف صحة. الحارس يستدعي `getAllAndOverride` بالمعالج ثم الصنف. دالة الصحة على صنف غير معلّم تبقى محمية إن لم تُعلَّم هي. خدمة غير موجودة في `providers` لا يراها الاكتشاف ولو لصق المزخرف.

## تمرين عبور

حارس مشترك بين HTTP ورسالة. ماذا يستقبل المتحكم، وماذا تستقبل الخدمة؟

<details>
<summary>الحل</summary>

المتحكم أو الحارس يقرأ `ExecutionContext`. الخدمة تستقبل `userId: string`. لا تمرر كائن Express إلى الدومين.

</details>

[Discovery و Metadata](../43-discovery-and-metadata/README.md)
