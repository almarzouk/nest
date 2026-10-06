# BeforeApplicationShutdown

## الهدف من هذا الدرس

استخدام اللحظة التي انتهت فيها كل دوال `onModuleDestroy`، والإشارة معروفة، واتصالات HTTP لم تُغلق بعد.

## المشكلة التي يحلها

بعض التنظيف يجب أن يرى سبب الإنهاء: `SIGTERM` من Kubernetes يختلف في قرارك عن `SIGINT` من الطرفية. وبعضه يجب أن يحدث قبل أن يقطع الإطار المقابس، مثل رفض عمل جديد أو تعليم readiness أن النسخة تغادر. `onModuleDestroy` لا يمرر الإشارة، و `onApplicationShutdown` يأتي بعد إغلاق الاتصالات.

## الفكرة ببساطة

`beforeApplicationShutdown(signal?)` يُستدعى بعد اكتمال معالجات `onModuleDestroy` كلها. عندما تنتهي وعوده، يستدعي الإطار إغلاق الاتصالات القائمة كجزء من `app.close()`. الخطاف جسر بين تنظيف الوحدات وقطع الباب.

## التعريف التقني

الواجهة `BeforeApplicationShutdown` تعرّف دالة تستقبل الإشارة نصًا عند الإنهاء بسبب إشارة. مع `app.close()` المباشر قد تصل القيمة فارغة لأن النداء ليس إشارة نظام.

التوثيق: بعد اكتمال المعالجات (الوعود حُسمت) تُغلق الاتصالات الموجودة. الاتجاه بين الوحدات هو اتجاه الإيقاف: من الجذر نحو الأوراق. الوعد يُنتظر قبل المتابعة.

Terminus، إن فعّلت shutdown hooks، يستخدم حدث دورة الحياة هذا ليبلغ `shutting_down`. ذلك تكامل لاحق في فصل الصحة، والفكرة هنا: الخطاف مكان مناسب لقول «لا ترسلوا عملًا جديدًا» قبل قطع المقابس.

على Windows دعم الإشارات محدود: `SIGINT` يعمل، و `SIGTERM` لا يصل لأن إنهاء العملية من مدير المهام غير قابل للاعتراض. الخطاف نفسه سليم عندما يبدأ الإغلاق.

## Mental Model

```text
كل onModuleDestroy اكتمل
  -> beforeApplicationShutdown(signal)
  -> app يغلق الاتصالات
  -> onApplicationShutdown(signal)
```

الإشارة معلومة. المقابس ما زالت هنا حتى يرجع الخطاف.

## أبسط مثال

```ts
import { Injectable, BeforeApplicationShutdown } from '@nestjs/common';

@Injectable()
export class DrainFlag implements BeforeApplicationShutdown {
  draining = false;

  beforeApplicationShutdown(signal?: string): void {
    this.draining = true;
    console.log(signal ?? 'close');
  }
}
```

بعد هذا السطر تستطيع خدمات ما زالت تعمل على طلبات قديمة أن تقرأ `draining` وتمتنع عن جدولة دفعة جديدة. الطلب الذي دخل قبل الإغلاق يكمله Node حتى يغلق الإطار السيرفر.

## شرح المثال

`signal` اختياري في النوع لأن `app.close()` قد لا يمرر `SIGTERM`. طباعة القيمة في التطوير تكفي لرؤية الفرق بين Ctrl+C ونداء الاختبار. العلم `draining` ذاكرة داخل Singleton. لا تضع العلم على خدمة `REQUEST` لأن تلك الأصناف لا تستقبل الخطاف.

## ماذا يحدث داخليًا؟

1. تنتهي سلسلة `onModuleDestroy`.
2. Nest يستدعي `beforeApplicationShutdown` ويمرر الإشارة إن وُجدت.
3. ينتظر كل الوعود.
4. يغلق الاتصالات القائمة. هذه خطوة الإطار بعد الخطاف، لا عملك داخل الدالة.
5. ثم `onApplicationShutdown`.
6. رفض الوعد يُحسم ثم تكمل السلسلة. لا تبتلع خطأ إغلاق مورد إلزامي بصمت إن كان التشخيص يهمك؛ سجّله ثم اترك السلسلة تكمل.

## مثال عملي

إيقاف مستهلك طابور قبل قطع HTTP، مع معرفة الإشارة:

```ts
import { Injectable, BeforeApplicationShutdown } from '@nestjs/common';
import { InvoiceWorker } from './invoice.worker.js';

@Injectable()
export class WorkerDrain implements BeforeApplicationShutdown {
  constructor(private readonly worker: InvoiceWorker) {}

  async beforeApplicationShutdown(signal?: string): Promise<void> {
    if (signal === 'SIGTERM') {
      await this.worker.stopAccepting();
    } else {
      await this.worker.stopAccepting();
    }
  }
}
```

في هذا المثال الفرعان متماثلان عمدًا: الإيقاف اللطيف مطلوب في الحالتين، وتسجيل `signal` يبقى للمراقبة. الفرق العملي يظهر عندما تريد مهلة أطول لـ `SIGTERM` فقط. اطلب تلك المهلة من المنسّق، ولا تنم داخل الخطاف دقيقة كاملة فتعطل مهلة القتل.

## أخطاء شائعة

- قراءة `request` هنا. الخطاف ليس داخل طلب.
- إغلاق `Prisma` مرة ثانية بعد `onModuleDestroy` فيتعطل الإغلاق.
- افتراض أن الاتصالات أُغلقت. الإغلاق يلي هذا الخطاف.
- نسيان أن Windows لن يسلم `SIGTERM`.
- حجز الخطاف بانتظار طويل فيتجاوز `terminationGracePeriodSeconds` وتُقتل العملية في منتصف الانتظار.

## متى نستخدمه؟

عندما تحتاج الإشارة، أو تريد ترتيبًا صريحًا بعد تدمير الوحدات وقبل قطع المقابس: علم الإفراغ، إيقاف قبول رسائل، إشعار جاهزية.

## متى لا نستخدمه؟

قطع اتصال قاعدة تملكه في `onModuleDestroy`. عمل أخير بعد زوال المقابس ينتظر `onApplicationShutdown`.

## مقارنة

| السؤال | الخطاف |
| --- | --- |
| أريد إغلاق مواردي أولًا | `onModuleDestroy` |
| أريد الإشارة قبل قطع المقابس | `beforeApplicationShutdown` |
| أريد أثرًا بعد إغلاق الاتصالات | `onApplicationShutdown` |

## تمارين

### تمرين سهل

ماذا يفعل الإطار عندما تنتهي وعود هذا الخطاف؟

<details>
<summary>الحل</summary>

يغلق الاتصالات القائمة، ثم يستدعي `onApplicationShutdown`.

</details>

### تمرين متوسط

لماذا لا يكفي `onModuleDestroy` لتمييز `SIGTERM`؟

<details>
<summary>الحل</summary>

لأن `onModuleDestroy` لا يستقبل وسيط الإشارة. الإشارة معامل في هذا الخطاف وفي `onApplicationShutdown`.

</details>

### تمرين متقدم

اختبار يستدعي `app.close()` ويرى `signal` بقيمة `undefined`. هل السلسلة مكسورة؟

<details>
<summary>الحل</summary>

لا. النداء الصريح ليس إشارة نظام، فالوسيط قد يغيب. السلسلة تعمل لأن `app.close()` يشغّل خطافات الإنهاء.

</details>

## أسئلة مراجعة

- أين يقع الخطاف بين destroy وإغلاق الاتصالات؟
- ماذا تستقبل الدالة؟
- لماذا المهلة الطويلة داخله خطرة في المنسّق؟

## الخلاصة

`beforeApplicationShutdown` بعد كل `onModuleDestroy` وقبل قطع الاتصالات. الإشارة متاحة، والباب لم يُغلق حتى يرجع الخطاف.

## ماذا بعد؟

`OnApplicationShutdown`: آخر خطوة بعد إغلاق الاتصالات.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
