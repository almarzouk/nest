# OnModuleInit

## الهدف من هذا الدرس

وضع التهيئة التي تحتاج اعتماديات محلولة في `onModuleInit`، وإبقاء `constructor` خاليًا من الشبكة والملفات.

## المشكلة التي يحلها

`constructor` يتلقى الاعتماديات ولا ينتظر `Promise` بشكل نظيف داخل حاوية Nest. فتح Prisma أو عميل HTTP هناك يخفي الفشل، وقد يكتمل الإقلاع بينما الاتصال ما زال معلقًا. `onModuleInit` يُستدعى بعد حل اعتماديات الوحدة المضيفة، وإن أرجعت الدالة وعدًا ينتظره الإطار قبل الانتقال.

## الفكرة ببساطة

عندما تنتهي الحاوية من بناء هذه الوحدة، تستدعي `onModuleInit` على الوحدة وعلى Providers و Controllers فيها. هذا أول خطاف إقلاع. مناسب لـ `$connect`، أو تحميل سرّ إلى ذاكرة الخدمة، أو فشل الإقلاع إن كان المورد الإلزامي غائبًا.

## التعريف التقني

الواجهة `OnModuleInit` من `@nestjs/common` تعرّف `onModuleInit(): void | Promise<void>`. التوثيق: تُستدعى مرة بعد حل اعتماديات الوحدة المضيفة.

الاستدعاء يسير من الوحدات الأعمق في رسم الاستيراد (والوحدات العامة) إلى الجذر. كل وحدة تُنتظر قبل التالية. لذلك وحدة قاعدة مستوردة بعمق تُنهى تهيئتها قبل وحدة الميزة التي تستوردها، وقبل `AppModule`.

الخطاف يعمل مع `app.listen()` و `app.init()` و `NestFactory.createApplicationContext()`. لا يعمل على صنف `Scope.REQUEST`.

## Mental Model

```text
حل اعتماديات الوحدة
  -> constructor (تعيين الحقول فقط)
  -> onModuleInit (اتصال، تحقق، انتظار)
  -> الوحدة التالية الأقرب إلى الجذر
```

إن رميت من `onModuleInit` يتوقف الإقلاع. هذا مرغوب لقاعدة إلزامية، وضار لفحص اختياري يمكن تأجيله.

## أبسط مثال

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }
}
```

```ts
import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service.js';

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

## شرح المثال

`PrismaClient` يُنشأ في `constructor` الافتراضي من غير فتح الشبكة. `onModuleInit` ينتظر `$connect`. بما أن `PrismaModule` ورقة في الرسم غالبًا، الاتصال يتم قبل `onModuleInit` في وحدات الكتب التي تحقن `PrismaService`. التصدير يتيح الحقن، ولا يستدعي الخطاف مرة ثانية.

## ماذا يحدث داخليًا؟

1. Nest يبني `PrismaService` ويحقن من تحتاجه هذه الوحدة.
2. يستدعي `onModuleInit` وينتظر الـ `Promise`.
3. إن نجح، ينتقل إلى الوحدة التالية في اتجاه الجذر.
4. `onApplicationBootstrap` لم يحن بعد، و `listen` لم يفتح المنفذ. فشل الاتصال يعني أن العملية لا تستقبل طلبات بنصف قاعدة.
5. الخدمات التي تعتمد على `PrismaService` يمكنها افتراض أن التهيئة في وحدة أعمق قد اكتملت إن كانت تلك الوحدة مستوردة أسفلها. لا تفترض ترتيب خدمتين شقيقتين داخل الوحدة نفسها.

## مثال عملي

تهيئة عميل ترفض الإقلاع إن غاب المتغير:

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class PaymentsClient implements OnModuleInit {
  constructor(private readonly config: ConfigService) {}

  async onModuleInit(): Promise<void> {
    const key = this.config.getOrThrow<string>('PAYMENTS_KEY');
    await this.ping(key);
  }

  private async ping(key: string): Promise<void> {
    void key;
  }
}
```

`getOrThrow` يرمي قبل الاستماع. المفتاح يبقى في الخدمة، و `constructor` لم يفتح شبكة. أغلق العميل لاحقًا في `onModuleDestroy`، لا في هذا الخطاف.

## أخطاء شائعة

- `await` داخل `constructor` عبر استدعاء من غير ربطه بخطاف، فيضيع الوعد.
- عمل ثقيل غير لازم للإقلاع، فيطول cold start.
- قراءة `REQUEST` هنا. لا يوجد طلب.
- نسيان أن الوعد المرفوض يوقف السلسلة. التقط الخطأ فقط إن كان المورد اختياريًا وتريد الإقلاع بدونه.
- استدعاء الخطاف يدويًا من اختبار بعد أن استدعاه الإطار، فيتصل مرتين.

## متى نستخدمه؟

اتصال إلزامي، تحقق من مخطط، أو تسخين ذاكرة محلية يجب أن ينجح قبل أن تكمل الوحدات الأعلى.

## متى لا نستخدمه؟

عمل يحتاج كل الوحدات مهيأة، مثل `ClientProxy.connect()` الذي يوثّقه Nest داخل `onApplicationBootstrap`. وكذلك أي منطق طلب.

## مقارنة

| المكان | ما يعرفه | هل يُنتظر؟ |
| --- | --- | --- |
| `constructor` | الاعتماديات المحقونة | لا تنتظر وعدًا عن الإطار |
| `onModuleInit` | اعتماديات هذه الوحدة محلولة | نعم |
| `onApplicationBootstrap` | كل الوحدات أُنهيت تهيئتها | نعم، وقبل الاستماع |

## تمارين

### تمرين سهل

ماذا يضمن الإطار إن كانت `onModuleInit` دالة `async`؟

<details>
<summary>الحل</summary>

ينتظر اكتمال الوعد أو رفضه قبل الانتقال إلى الوحدة التالية في سلسلة الإقلاع.

</details>

### تمرين متوسط

لماذا `$connect` أنسب هنا من `constructor`؟

<details>
<summary>الحل</summary>

لأن الاتصال غير متزامن ويجب أن يفشل الإقلاع إن فشل، بعد أن تكون الاعتماديات جاهزة. `constructor` للتعيين.

</details>

### تمرين متقدم

`BooksService` و `PrismaService` في الوحدة نفسها. هل يجوز أن تفترض `BooksService.onModuleInit` أن `$connect` انتهى؟

<details>
<summary>الحل</summary>

لا. الترتيب الموثق بين الوحدات حسب البعد عن الجذر، لا بين أصناف الوحدة الواحدة. انقل الاتصال إلى وحدة أعمق تُستورد، أو اجعل الكتب تتصل عبر الخدمة عند أول استخدام بعد أن تكون وحدة القاعدة ورقة في الرسم.

</details>

## أسئلة مراجعة

- متى يُستدعى `onModuleInit` نسبة إلى حل الاعتماديات؟
- أي وحدات تُستدعى أولًا؟
- لماذا لا يصلح لصنف Request-scoped؟

## الخلاصة

`onModuleInit` هو مكان التهيئة غير المتزامنة بعد حقن الوحدة. انتظر الوعد، وافشل الإقلاع إن كان المورد شرطًا للاستماع.

## ماذا بعد؟

`OnApplicationBootstrap`: ما الذي ينتظر اكتمال كل الوحدات.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
