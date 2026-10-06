# OnApplicationBootstrap

## الهدف من هذا الدرس

تشغيل ما يحتاج رسم الوحدات كاملًا، مع بقاء المنفذ مغلقًا حتى ينتهي هذا الخطاف.

## المشكلة التي يحلها

`onModuleInit` مبكر لبعض العمل: عميل Microservice ما زال كسولًا، أو مهمة تحتاج Provider من وحدة شقيقة لم تُرتَّب معك في العمق نفسه. إن فتحت `listen` ثم بدأت هذا العمل، يصل طلب قبل أن يكتمل التسخين. `onApplicationBootstrap` يقع بعد تهيئة كل الوحدات وقبل الاستماع.

## الفكرة ببساطة

بعد أن رأى كل Module خطاف البداية الخاص به، يمر Nest مرة ثانية بـ `onApplicationBootstrap` بالاتجاه نفسه: من الأوراق إلى الجذر، مع انتظار كل وحدة. ثم يبدأ الاستماع. الخطاف يقول: الرسم جاهز، والباب لم يُفتح.

## التعريف التقني

الواجهة `OnApplicationBootstrap` تعرّف `onApplicationBootstrap(): void | Promise<void>`. التوثيق: تُستدعى مرة بعد تهيئة كل الوحدات، وقبل الاستماع للاتصالات.

توثيق Microservices يضع `ClientProxy.connect()` هنا عندما تريد تأخير الإقلاع حتى يقوم الاتصال، لأن `ClientProxy` كسول ولا يتصل عند أول بناء. استدعاء `connect()` داخل هذا الخطاف يرفض الإقلاع إن كان الناقل غائبًا.

الخطاف لا يعمل على أصناف `Scope.REQUEST`. و `app.close()` لا يستدعيه؛ له سلسلة إيقاف منفصلة.

## Mental Model

```text
كل onModuleInit اكتمل
  -> onApplicationBootstrap من الأوراق إلى الجذر
  -> listen / startAllMicroservices
```

ما تضعه هنا يحجز cold start. ضيفه إن كان شرطًا لصحة أول طلب، وأجّله إلى أول استخدام إن كان اختياريًا.

## أبسط مثال

```ts
import { Injectable, OnApplicationBootstrap } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';

@Injectable()
export class BillingGateway implements OnApplicationBootstrap {
  constructor(private readonly client: ClientProxy) {}

  async onApplicationBootstrap(): Promise<void> {
    await this.client.connect();
  }
}
```

حتى يكتمل `connect` لا يستدعي الإطار `listen`. طلب HTTP لا يجد عملية تقبل المقابس قبل ذلك.

## شرح المثال

`ClientProxy` يُحقن كاعتماد عادي، والاتصال الفعلي مؤجّل. الخطاف يجعل الفشل ظاهرًا أثناء الإقلاع. أغلق العميل في خطاف الإيقاف المناسب، لأن هذا الملف يفتح فقط. إن كان الناقل اختياريًا للمنتج، فالانتظار هنا يوقف المتجر كله لأجل تكامل جانبي، وذلك قرار منتج لا تفصيل إطار.

## ماذا يحدث داخليًا؟

1. تنتهي سلسلة `onModuleInit` لكل الوحدات.
2. تبدأ سلسلة `onApplicationBootstrap` بالترتيب نفسه: الأعمق ثم الأقرب للجذر، مع `await`.
3. بعد آخر وحدة يستدعي Nest مسار الاستماع: `app.listen` أو ما يعادله في `createMicroservice`.
4. رفض الوعد يمنع الاستماع. العملية تتوقف حسب `abortOnError` في مسار HTTP المعتاد.
5. لا يوجد طلب، فلا `REQUEST` ولا مستخدم.

## مثال عملي

تسخين ذاكرة قراءة بعد أن أصبحت خدمات الكتالوج قابلة للحقن:

```ts
import { Injectable, OnApplicationBootstrap } from '@nestjs/common';
import { CatalogReader } from './catalog.reader.js';

@Injectable()
export class CatalogWarmup implements OnApplicationBootstrap {
  constructor(private readonly reader: CatalogReader) {}

  async onApplicationBootstrap(): Promise<void> {
    await this.reader.preloadGenres();
  }
}
```

`preloadGenres` يقرأ من قاعدة فُتحت في `onModuleInit` لوحدة أعمق. وضعه في `onModuleInit` الخاص بهذه الخدمة يغامر بترتيب الأشقاء. وضعه بعد `listen` يسمح لأول `GET /genres` أن يسبق التسخين. الخطاف الأوسط يمنع الاثنين.

## أخطاء شائعة

- فتح منفذ يدويًا داخل الخطاف فيتكرر الاستماع مع `listen`.
- اتصال كسول لا يلزم أن يحجز الإقلاع، فيطول زمن النسخة في Kubernetes.
- افتراض أن Controllers استقبلت طلبًا. الاستماع لم يبدأ.
- تسجيل `setInterval` هنا ونسيان إيقافه، فتبقى العملية بعد `app.close()`.
- استخدامه على Gateway أو استراتيجية يجب أن تبقى Singleton ثم وسم الصنف `REQUEST`. الخطاف لن يعمل.

## متى نستخدمه؟

اتصال ناقل يجب أن ينجح قبل الباب، أو تسخين يعتمد على رسم مكتمل، أو فحص جاهزية داخلي تريد فشله قبل أن يراك المنسّق.

## متى لا نستخدمه؟

اتصال يخص وحدة واحدة ولا يحتاج الجيران: `onModuleInit`. تنظيف: خطافات الإيقاف. عمل كل طلب: مسار الطلب.

## مقارنة

| الخطاف | الرسم | الباب |
| --- | --- | --- |
| `onModuleInit` | وحدتك محلولة، وحدات أعلى قد لا تكون | مغلق |
| `onApplicationBootstrap` | كل الوحدات هُيئت | ما زال مغلقًا حتى ينتهي |
| أول طلب | الرسم حي | مفتوح |

## تمارين

### تمرين سهل

هل يصل `GET /health` أثناء `onApplicationBootstrap`؟

<details>
<summary>الحل</summary>

لا. الخطاف يُستدعى قبل الاستماع للاتصالات.

</details>

### تمرين متوسط

أين يوثّق Nest نداء `ClientProxy.connect()` إن أردت حجز الإقلاع حتى يتصل العميل؟

<details>
<summary>الحل</summary>

داخل `onApplicationBootstrap`. العميل كسول، والخطاف يؤخر متابعة الإقلاع حتى يكتمل `connect`.

</details>

### تمرين متقدم

مهمة cron اختيارية تتصل ببريد خارجي. لماذا تبقيها خارج هذا الخطاف في خدمة المتجر الأساسية؟

<details>
<summary>الحل</summary>

لأن الوعد المرفوض يوقف الاستماع. تكامل اختياري لا يجوز أن يمنع `listen`. ابدأه بعد الإقلاع أو التقط الفشل داخله إن قبلت العمل من دونه.

</details>

## أسئلة مراجعة

- ما الفرق الزمني عن `onModuleInit`؟
- هل يُنتظر الـ `Promise`؟
- ماذا يحدث للباب إن رمى الخطاف؟

## الخلاصة

`onApplicationBootstrap` يعني أن كل الوحدات أُنهيت تهيئتها وأن المنفذ لم يُفتح بعد. ضع فيه ما يجب أن ينجح قبل أول اتصال خارجي.

## ماذا بعد؟

`OnModuleDestroy`: أول خطاف في مرحلة الإنهاء.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
- NestJS — Microservices (ClientProxy.connect): https://docs.nestjs.com/microservices/basics
