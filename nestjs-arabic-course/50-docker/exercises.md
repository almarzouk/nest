# تمارين الفصل

## سهل

ماذا تضع مكان المضيف في `REDIS_URL` لخدمة اسمها `redis` على شبكة Compose؟

<details>
<summary>الحل</summary>

`redis`، مثل `redis://redis:6379`. `localhost` هو حاوية التطبيق.

</details>

## متوسط

لماذا تفشل `npm run start:dev` في صورة المرحلة `runner`؟

<details>
<summary>الحل</summary>

الصورة بلا مصدر `src` وبلا اعتماديات التطوير. الأمر الإنتاجي `node dist/main.js`.

</details>

## متقدم

الحاوية تُقتل بعد عشر ثوانٍ من `stop` و Postgres يبقى عليه جلسات. ماذا ينقص في `main.ts` وماذا تفعل الخدمات؟

<details>
<summary>الحل</summary>

`enableShutdownHooks()` حتى يصل `SIGTERM`. `PrismaService` وعميل Redis يغلقان في `onModuleDestroy`. صيغة `CMD` التنفيذية حتى لا يبتلع غلاف الإشارة. أزل المؤقتات التي تبقي العملية حية.

</details>
