# تفعيل Shutdown Hooks

## الهدف من هذا الدرس

جعل `SIGTERM` و `SIGINT` يشغّلان سلسلة الإنهاء، ومعرفة لماذا المستمعون مطفأون افتراضيًا.

## المشكلة التي يحلها

استدعاء `app.close()` في اختبار يشغّل الإيقاف. إيقاف الحاوية يرسل إشارة إلى العملية، وتلك الإشارة لا تصل إلى `onModuleDestroy` و `beforeApplicationShutdown` و `onApplicationShutdown` إلا إذا ناديت `enableShutdownHooks()` عند الإقلاع. من غيرها تُقتل العملية والعملاء ما زالوا مفتوحين.

## الفكرة ببساطة

المستمعون يستهلكون مواردًا، لذلك Nest يتركهم مطفأين. سطر واحد بعد `NestFactory.create` يبدأ الاستماع لإشارات الإيقاف. في الإنتاج تريده. في عملية واحدة تشغّل عشرات تطبيقات الاختبار قد يحذّر Node من كثرة المستمعين، ولهذا ليس الافتراض.

## التعريف التقني

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks();
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

بعدها، إشارة مدعومة تستدعي الخطافات الثلاثة بالترتيب، وتمرر الإشارة إلى الأخيرين. إن أرجعت دالة وعدًا، لا يكمل الإطار حتى يُحسم.

توثيق المنصة: على Windows يعمل `SIGINT`، وكذلك `SIGBREAK` وبدرجة `SIGHUP`. `SIGTERM` لا يصل لأن القتل من مدير المهام غير مشروط. التفاصيل في توثيق إشارات libuv وNode.

تلميح التوثيق: تشغيل عدة تطبيقات Nest في عملية Node واحدة قد يطبع تحذير `MaxListenersExceeded`. أبقِ المستمعين حيث توجد نسخة إنتاج واحدة، وراعِ الاختبارات المتوازية.

`enableShutdownHooks()` لا يغني عن كتابة الخطافات. هو يفتح الطريق للإشارة فقط. `app.close()` يعمل من دونه.

Terminus ينصح بتفعيل المستمعين كي يبلغ `shutting_down` ويحترم `gracefulShutdownTimeoutMs`. ذلك فصل الصحة. هنا يكفي أن الإشارة أصبحت قادرة على بدء السلسلة.

## Mental Model

```text
من غير التفعيل: SIGTERM -> العملية تموت والخطافات صامتة
مع التفعيل:    SIGTERM -> destroy -> before -> إغلاق المقابس -> shutdown
دائمًا:        app.close() -> السلسلة نفسها
```

التفعيل مستمعون. الخطافات هي العمل.

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();
await app.listen(3000);
```

ثم في خدمة Singleton سجّلت `onModuleDestroy`. Ctrl+C في الطرفية يرسل `SIGINT`، فترى خطاف الإيقاف على أنظمة تدعم هذه الإشارة، ومنها macOS وLinux.

## شرح المثال

النداء قبل `listen` حتى تكون المستمعات جاهزة منذ فتح الباب. تأخيره إلى ما بعد أول طلب يترك نافذة تُقتل فيها العملية بلا تنظيف. لا تستدعِ الدالة داخل Provider لكل طلب. مرة في `bootstrap` تكفي للتطبيق كله.

## ماذا يحدث داخليًا؟

1. `enableShutdownHooks` يسجّل مستمعي إشارات على العملية.
2. عند الإشارة المدعومة يبدأ Nest مسار `close` الداخلي.
3. تُستدعى خطافات الإنهاء بالترتيب الموثق، مع انتظار الوعود.
4. تُغلق اتصالات السيرفر بين `beforeApplicationShutdown` و `onApplicationShutdown`.
5. العملية تخرج عندما لا تبقى مقابض. المؤقت الذي لم يُمسح يمنع الخروج حتى بعد السلسلة.
6. على Windows، `SIGTERM` لا يدخل هذا المسار.

## مثال عملي

اختبار يغلق صراحة، وعملية إنتاج تفعّل المستمعين:

```ts
import { Test } from '@nestjs/testing';
import { AppModule } from './app.module.js';

const testingModule = await Test.createTestingModule({
  imports: [AppModule],
}).compile();
const app = testingModule.createNestApplication();
await app.init();
await app.close();
```

اختبار `close` لا يحتاج `enableShutdownHooks` لرؤية `onModuleDestroy`. صورة Docker التي تستلم `SIGTERM` من المنسّق تحتاجه في `main.ts`. اخلط المسارين حسب البيئة، ولا تحذف السطر من الإنتاج لأن الاختبار استغنى عنه.

## أخطاء شائعة

- نسيان السطر ثم لوم `onModuleDestroy`.
- تفعيله في كل عامل ضمن عملية اختبار واحدة حتى يفيض عداد المستمعين.
- افتراض `SIGTERM` على Windows.
- الاعتقاد أن التفعيل يغلق Prisma. التفعيل يبدأ السلسلة فقط.
- استدعاء `process.exit` داخل الخطاف قبل `await` فيقطع الانتظار الذي وفره الإطار.

## متى نستخدمه؟

في `bootstrap` لأي عملية طويلة ستُوقف بإشارة: حاوية، منصة PaaS، `node dist/main.js` خلف systemd.

## متى لا نستخدمه؟

لا تكرره لكل وحدة. ولا تعتمد عليه وحده في اختبار يستدعي `app.close()`. سكربت قصير ينتهي من نفسه قد لا يحتاج مستمعي إشارات إن لم يرسل إليه أحد `SIGTERM`.

## مقارنة

| المسار | هل يشغّل خطافات الإنهاء؟ |
| --- | --- |
| `app.close()` / `app.init` ثم `close` | نعم، من غير تفعيل |
| `SIGTERM` بعد `enableShutdownHooks()` | نعم على الأنظمة التي توصّل الإشارة |
| `SIGTERM` من غير التفعيل | الخطافات لا تُستدعى |
| قتل قاسٍ لا يسلّم إشارة | لا خطاف |

## تمارين

### تمرين سهل

أين تُستدعى `enableShutdownHooks`؟

<details>
<summary>الحل</summary>

مرة في `bootstrap` بعد `NestFactory.create` وقبل `listen`.

</details>

### تمرين متوسط

لماذا المستمعون مطفأون افتراضيًا؟

<details>
<summary>الحل</summary>

لأنهم يستهلكون مواردًا. عدة تطبيقات Nest في عملية واحدة قد تطلق تحذير كثرة المستمعين في Node.

</details>

### تمرين متقدم

حاوية Linux تستلم `SIGTERM` وPrisma يبقى متصلًا. ما الشرطان الناقصان المحتملان؟

<details>
<summary>الحل</summary>

إما لم تُستدعَ `enableShutdownHooks()`، أو لا يوجد `onModuleDestroy` ينفذ `$disconnect`. التفعيل من غير خطاف لا يغلق العميل، والخطاف من غير تفعيل لا يراه `SIGTERM`.

</details>

## أسئلة مراجعة

- ماذا يغيّر التفعيل في إشارات النظام؟
- هل الاختبار الذي ينادي `close` يحتاجه؟
- ما حد Windows مع `SIGTERM`؟

## الخلاصة

`enableShutdownHooks()` يفتح أذن العملية لإشارات الإيقاف. اكتبه في إقلاع الإنتاج، واكتب الخطافات التي تحرّر الموارد، وتذكر أن `app.close()` مسار مستقل يعمل من دونه.

## ماذا بعد؟

أمثلة تجمع الخطافات على Prisma ومؤقت وعميل رسائل.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
