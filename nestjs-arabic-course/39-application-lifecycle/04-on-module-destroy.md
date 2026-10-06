# OnModuleDestroy

## الهدف من هذا الدرس

إغلاق ما فتحته الوحدة عندما يبدأ الإنهاء، قبل خطافَي الإيقاف اللذين يأتيان بعده.

## المشكلة التي يحلها

العملية التي تستلم `SIGTERM` وتُقتل فورًا تترك اتصال Prisma أو مؤقتًا أو عميلًا مفتوحًا. `onModuleDestroy` هو أول خطاف terminating. مكانه تحرير موارد هذه الوحدة، بشرط أن يكون الإغلاق قد بدأ فعلًا عبر `app.close()` أو عبر إشارة بعد `enableShutdownHooks()`.

## الفكرة ببساطة

عندما يدخل التطبيق مرحلة الإنهاء، يستدعي Nest `onModuleDestroy` من الجذر نحو الأوراق، وينتظر كل وحدة. الدالة لا تستقبل اسم الإشارة. بعدها، لا قبلها، يأتي `beforeApplicationShutdown`.

## التعريف التقني

الواجهة `OnModuleDestroy` تعرّف `onModuleDestroy(): void | Promise<void>`. التوثيق: تُستدعى بعد استلام إشارة إنهاء مثل `SIGTERM`. المسار الآخر هو `app.close()` الصريح، وهذا يعمل حتى من غير تفعيل مستمعي الإشارات.

الاتجاه عكس الإقلاع: الوحدات الأقل عمقًا أولًا، والأوراق آخرًا. الوعد يُنتظر، والرفض لا يوقف السلسلة عن الاستمرار حتى يُحسم الوعد (resolved أو rejected) ثم يكمل الإطار.

أصناف الطلب لا تستقبل هذا الخطاف. `app.close()` يشغّل السلسلة ولا ينهي عملية Node إن بقيت مقابض مفتوحة.

## Mental Model

```text
قرار إنهاء
  -> onModuleDestroy من الجذر إلى الأوراق
  -> beforeApplicationShutdown
  -> إغلاق الاتصالات القائمة
  -> onApplicationShutdown
```

أغلق هنا موردًا تملكه الوحدة: عميل قاعدة، مؤقت، اشتراك. لا تفترض أن مقابس HTTP أُغلقت؛ ذلك يقع بعد `beforeApplicationShutdown`.

## أبسط مثال

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

الزوج `connect` / `disconnect` يبقى على الخدمة نفسها. الإغلاق ينتظر انتهاء الاستعلامات التي يسمح Prisma بإنهائها قبل قطع العميل.

## شرح المثال

`onModuleInit` يفتح و `onModuleDestroy` يغلق. من غير مسار إنهاء حقيقي لن يُستدعى الإغلاق: اختبار يستدعي `app.close()`، أو عملية إنتاج فعّلت `enableShutdownHooks` ثم استلمت الإشارة. استدعاء `$disconnect` من غير الخطاف في نهاية طلب عادي يغلق المجمع تحت أقدام الطلبات الأخرى لأن الخدمة Singleton.

## ماذا يحدث داخليًا؟

1. `app.close()` أو الإشارة المفعّلة تبدأ terminating.
2. Nest يمشي الوحدات من الأقرب للجذر إلى الأعمق.
3. لكل وحدة يستدعي `onModuleDestroy` وينتظر.
4. خدمة الكتب في الجذر يمكن أن تتوقف عن جدولة عمل جديد قبل أن تُقطع ورقة قاعدة البيانات.
5. بعد اكتمال الجميع يأتي `beforeApplicationShutdown(signal)`. هذه الدالة لا ترى `signal`.
6. ثم تُغلق الاتصالات، ثم `onApplicationShutdown`.

## مثال عملي

مؤقت داخل العملية يجب أن يُمسح وإلا بقيت العملية حية بعد `close`:

```ts
import { Injectable, OnModuleDestroy, OnApplicationBootstrap } from '@nestjs/common';

@Injectable()
export class InvoiceSweep implements OnApplicationBootstrap, OnModuleDestroy {
  private timer: NodeJS.Timeout | undefined;

  onApplicationBootstrap(): void {
    this.timer = setInterval(() => {
      void this.sweep();
    }, 60_000);
  }

  onModuleDestroy(): void {
    if (this.timer) {
      clearInterval(this.timer);
    }
  }

  private async sweep(): Promise<void> {
    return undefined;
  }
}
```

`clearInterval` هنا يحرر المقبض قبل خطوات الإيقاف التالية. إن تركته، `app.close()` ينهي خطافات Nest وتبقى عملية Node.

## أخطاء شائعة

- انتظار `SIGTERM` من غير `enableShutdownHooks`، فالخطاف لا يعمل.
- إغلاق مجمع قاعدة في كل طلب لأن الخدمة ظُنت per-request.
- افتراض وصول `signal` وسيطًا. التوقيع بلا معاملات.
- إيقاف HTTP يدويًا هنا بينما الإطار سيغلق الاتصالات لاحقًا.
- رمي خطأ ثم افتراض أن بقية الوحدات لن تُستدعى. الإطار يكمل بعد حسم الوعد.

## متى نستخدمه؟

تحرير مورد ملكته عند الإقلاع: disconnect، إيقاف مؤقت، إغلاق عميل تملكه هذه الوحدة.

## متى لا نستخدمه؟

منطق يعتمد على اسم الإشارة مثل `SIGINT` مقابل `SIGTERM`: ذلك وسيط الخطافين التاليين. رد readiness أثناء فترة السماح يخص Terminus مع خطافات الإيقاف، لا هذا التوقيع وحده.

## مقارنة

| الخطاف | الإشارة | موقعه في الإيقاف |
| --- | --- | --- |
| `onModuleDestroy` | لا يستقبلها | الأول |
| `beforeApplicationShutdown` | وسيط أول | بعد كل destroy، وقبل إغلاق الاتصالات |
| `onApplicationShutdown` | وسيط أول | بعد إغلاق الاتصالات |

## تمارين

### تمرين سهل

هل توقيع `onModuleDestroy` يأخذ `signal`؟

<details>
<summary>الحل</summary>

لا. الإشارة تصل إلى `beforeApplicationShutdown` و `onApplicationShutdown` فقط.

</details>

### تمرين متوسط

اختبار يستدعي `app.close()` من غير `enableShutdownHooks`. هل يعمل `onModuleDestroy`؟

<details>
<summary>الحل</summary>

نعم. `app.close()` يشغّل خطافات الإنهاء. تفعيل المستمعين لازم لإشارات النظام، لا للنداء الصريح.

</details>

### تمرين متقدم

لماذا يُستدعى الجذر قبل ورقة `PrismaModule` في هذا الخطاف؟

<details>
<summary>الحل</summary>

الإيقاف يعكس الإقلاع. التوثيق: من الأقل عمقًا إلى الأكثر عمقًا، حتى تتوقف الوحدات العليا عن الاستخدام قبل قطع الأوراق.

</details>

## أسئلة مراجعة

- ما الشرطان اللذان يبدآن هذا الخطاف؟
- ماذا تفعل بوعده؟
- لماذا `clearInterval` جزء من الإغلاق الصحيح؟

## الخلاصة

`onModuleDestroy` أول خطوة إنهاء، بلا اسم إشارة، ومن الجذر نحو الأوراق. أغلق فيه ما فتحته، واترك إغلاق المقابس للإطار بعد الخطاف التالي.

## ماذا بعد؟

`BeforeApplicationShutdown`: ماذا تستطيع أن تفعل والإشارة معروفة والاتصالات ما زالت قائمة.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
