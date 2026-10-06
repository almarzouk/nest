# OnApplicationShutdown

## الهدف من هذا الدرس

تنفيذ آخر عمل بعد أن أغلق الإطار الاتصالات القائمة، وقبل أن تعتبر `app.close()` منتهية.

## المشكلة التي يحلها

مورد يُغلق بينما المقابس ما زالت تكتب يترك طلبًا في نصف رد. ومورد تنتظره بعد قطع الباب، مثل حذف ملف قفل أو إغلاق عميل لم يعد يخدم طلبًا، يحتاج آخر خطاف. `onApplicationShutdown` هو ذلك المكان: بعد إغلاق الاتصالات، وآخر خطوة في `app.close()`.

## الفكرة ببساطة

الإشارة تصل وسيطًا كما في الخطاف السابق. الفرق الزمني: الاتصالات أُغلقت. لا طلب HTTP ما زال على مقبس التطبيق. ما تبقى هو عملية Node وقد تبقى حية إن تركت مؤقتًا أو مقبضًا.

## التعريف التقني

الواجهة `OnApplicationShutdown` تعرّف `onApplicationShutdown(signal?: string): void | Promise<void>`. التوثيق: تُستدعى بعد إغلاق الاتصالات، آخر خطوة من `app.close()`.

مثال التوثيق:

```ts
import { Injectable, OnApplicationShutdown } from '@nestjs/common';

@Injectable()
export class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string): void {
    console.log(signal);
  }
}
```

القيمة المطبوعة مثل `SIGINT`. الاتجاه بين الوحدات يبقى اتجاه الإيقاف. أصناف `Scope.REQUEST` لا تُستدعى. إن لم تنادِ `app.close()` ولم تفعّل المستمعين، لن ترى الدالة إشارة النظام.

`app.close()` لا يستدعي `process.exit`. المؤقتات والمهام الخلفية تبقي العملية. هذا الخطاف آخر فرصة لإسقاط تلك المقابض إن لم تُسقط في `onModuleDestroy`.

## Mental Model

```text
الاتصالات أُغلقت
  -> onApplicationShutdown(signal)
  -> app.close() يُعتبر مكتملًا
  -> العملية تخرج فقط إن لم يبقَ مقبض مفتوح
```

لا تبدأ قبول عمل جديد هنا. الباب أُغلق.

## أبسط مثال

```ts
import { Injectable, OnApplicationShutdown } from '@nestjs/common';
import { createWriteStream, type WriteStream } from 'node:fs';

@Injectable()
export class AuditFile implements OnApplicationShutdown {
  private stream: WriteStream | undefined;

  open(): void {
    this.stream = createWriteStream('/tmp/audit.log', { flags: 'a' });
  }

  async onApplicationShutdown(signal?: string): Promise<void> {
    const stream = this.stream;
    if (!stream) {
      return;
    }
    await new Promise<void>((resolve) => stream.end(resolve));
    void signal;
  }
}
```

إغلاق الملف بعد صمت المقابس يمنع كتابة سطر جزئي من طلب كان ما زال يُرسل.

## شرح المثال

`end` يعيد وعدًا ينتظره الإطار لأن الدالة `async`. `signal` متاح للسجل إن أردت تمييز Ctrl+C عن إيقاف المنسّق. الملف مثال مورد عملية، لا مورد طلب. افتحه في خطاف إقلاع، وأغلقه هنا إن كان يجب أن يعيش حتى آخر لحظة بعد قطع HTTP.

## ماذا يحدث داخليًا؟

1. تكتمل `beforeApplicationShutdown`.
2. الإطار يغلق سيرفر المنصة والاتصالات القائمة.
3. يستدعي `onApplicationShutdown` ويمرر الإشارة إن وُجدت.
4. ينتظر الوعود. بعدها تكتمل `app.close()`.
5. عملية Node تبقى إن بقي `setInterval` أو اتصال لم يُغلق. راقب ذلك في الاختبار: `close` يرجع والعملية لا تخرج.
6. على Windows قد ترى `SIGINT` ولا ترى `SIGTERM`.

## مثال عملي

سجل ختامي يُكتب مرة واحدة بعد الهدوء:

```ts
import { Injectable, Logger, OnApplicationShutdown } from '@nestjs/common';

@Injectable()
export class ShutdownNote implements OnApplicationShutdown {
  private readonly logger = new Logger(ShutdownNote.name);

  onApplicationShutdown(signal?: string): void {
    this.logger.log(`connections closed (${signal ?? 'app.close'})`);
  }
}
```

السطر يعني أن مرحلة المقابس انتهت. لا تضع `$disconnect` الوحيد هنا إن كانت وحدات أخرى ما زالت تحتاجه أثناء `onModuleDestroy`؛ تلك الدوال تُستدعى قبل هذا الخطاف، فقطع القاعدة هنا متأخر عنها. إن كانت الورقة الأخيرة في الرسم هي الوحيدة التي تملك العميل، فالإغلاق في `onModuleDestroy` لتلك الورقة يسبق هذا الخطاف أصلًا.

## أخطاء شائعة

- اعتبار الخطاف قاتلًا للعملية. `close` لا يستدعي `process.exit`.
- فتح اتصال جديد هنا بعد أن توقفت عن قبول العمل.
- توقع تشغيله على نسخة Request-scoped.
- قياس جاهزية Kubernetes بهذا الخطاف. الجاهزية تُسأل عبر HTTP قبل الإيقاف، وTerminus يبلغ `shutting_down` أبكر.
- نسيان `enableShutdownHooks` ثم استغراب صمت الدالة عند `SIGTERM`.

## متى نستخدمه؟

آخر تحرير بعد زوال الاتصالات: إغلاق ملف، سجل ختامي، إسقاط مقبض يبقي العملية حية.

## متى لا نستخدمه؟

قطع مورد تحتاجه وحدات أثناء `onModuleDestroy`. قرار «ما زلت أقبل الطلبات» يحدث قبل إغلاق الاتصالات، في `beforeApplicationShutdown`.

## مقارنة

| بعد هذا الخطاف | ما الذي تغيّر |
| --- | --- |
| عن `beforeApplicationShutdown` | الاتصالات أُغلقت |
| عن خروج العملية | `app.close()` اكتمل، وNode قد يبقى |
| عن طلب حي | لا مقبس تطبيق يقبل ردًا جديدًا |

## تمارين

### تمرين سهل

ما وسيط `onApplicationShutdown`؟

<details>
<summary>الحل</summary>

الإشارة نصًا، مثل `SIGINT`، وقد تغيب مع `app.close()` الصريح.

</details>

### تمرين متوسط

لماذا تبقى العملية بعد أن رجع `await app.close()`؟

<details>
<summary>الحل</summary>

لأن `close` يشغّل السلسلة ويغلق سيرفر الإطار ولا يستدعي إنهاء العملية. مؤقت أو مقبض مفتوح يبقي Node حيًا.

</details>

### تمرين متقدم

أين تغلق `Prisma` إن كانت وحدات أخرى تستدعيه داخل `onModuleDestroy`؟

<details>
<summary>الحل</summary>

في `onModuleDestroy` لوحدة القاعدة لأنها ورقة وتُستدعى آخرًا في الإيقاف، بعد أن أنهت الوحدات الأعلى عملها. إغلاقه أول مرة في `onApplicationShutdown` يأتي بعد تلك الدوال، وقد تكون حاولت الاستخدام أثناء destroy.

</details>

## أسئلة مراجعة

- ما الذي أُغلق قبل هذا الخطاف؟
- هل الخطاف ينهي العملية؟
- أي أصناف لا تراه؟

## الخلاصة

`onApplicationShutdown` آخر خطوة في `app.close()` بعد إغلاق الاتصالات. حرر فيه ما يجب أن يبقى حتى تلك اللحظة، وأسقط المقابض التي تمنع خروج Node.

## ماذا بعد؟

تفعيل Shutdown Hooks حتى تصل إشارات النظام إلى هذه السلسلة.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
