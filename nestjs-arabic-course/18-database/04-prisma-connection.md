# Prisma: الاتصال

## الهدف من هذا الدرس

فتح Prisma Client على PostgreSQL من مزوّد Nest، وربط العمر بخطافات الوحدة.

## المشكلة التي يحلها

إنشاء `PrismaClient` داخل كل خدمة يفتح مجمعات متعددة ويمنع إغلاقها عند الإيقاف. قراءة `DATABASE_URL` من `process.env` في أعلى الملف تسبق التحقق. أول استعلام كسول قد يفشل بعد أن صار الخادم يستقبل طلبات.

## الفكرة ببساطة

`PrismaService` يمتد `PrismaClient`، يبني محوّل PostgreSQL من العنوان الذي يقدّمه `ConfigService`، ويتصل في `onModuleInit` ويقطع في `onModuleDestroy`. نسخة واحدة تُحقن حيث نحتاج القاعدة.

## التعريف التقني

توثيق Nest الحالي يستخدم Prisma ORM 7. ثبّت الإصدار صراحة لأن وسم `latest` وقت كتابة تلك الصفحة قد يشير إلى إصدار سابق للطرح لا يوفّر أوامر `migrate` و `generate` نفسها:

```bash
npm install prisma@7 dotenv --save-dev
npm install @prisma/client@7 @prisma/adapter-pg
```

Prisma 7 يحتاج driver adapter، أو عنوان Accelerate، لبناء العميل. لـ PostgreSQL الصنف `PrismaPg` من `@prisma/adapter-pg`.

```ts
const adapter = new PrismaPg({
  connectionString: configService.getOrThrow<string>("DATABASE_URL"),
});
super({ adapter });
```

العميل المولَّد يُستورد من المسار الذي يحدده `output` في المولّد، ومع ESM بلاحقة `.js`:

```ts
import { PrismaClient } from "../generated/prisma/client.js";
```

`$connect()` في `onModuleInit` يفشل الإقلاع إن كان العنوان خاطئًا، بدل الانتظار الكسول لأول استعلام. `$disconnect()` في `onModuleDestroy` يغلق المجمع. Nest يستدعي الخطاف عند `app.close()`. لتفعيله عند `SIGTERM` استدعِ `app.enableShutdownHooks()` في `main.ts`.

Prisma 7 لا يعتمد حدث `beforeExit` الذي استخدمته تكاملات أقدم.

عنوان PostgreSQL يعيش في `DATABASE_URL`. ملف تهيئة Prisma 7 يقرأه لأوامر CLI. التطبيق يقرأه عبر `ConfigModule` كما في الصفحة الرسمية.

## Mental Model

```text
ConfigService DATABASE_URL
        |
 PrismaPg adapter
        |
 PrismaService extends PrismaClient
        |
 onModuleInit $connect -> مجمع -> PostgreSQL
 onModuleDestroy $disconnect
```

## أبسط مثال

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../generated/prisma/client.js";

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor(config: ConfigService) {
    const adapter = new PrismaPg({
      connectionString: config.getOrThrow<string>("DATABASE_URL"),
    });
    super({ adapter });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

## شرح المثال

`super` يُستدعى بعد بناء المحوّل لأن الصنف الأب يحتاج `adapter` في الخيارات. `getOrThrow` يوقف الإقلاع إن غاب العنوان، وهذا يتفق مع مخطط البيئة إن كان `DATABASE_URL` مطلوبًا.

الخدمة ما زالت تحتاج تسجيلًا في وحدة وتصديرًا. درس «وضع Prisma داخل Module» يفصل ذلك. هنا الحد هو الاتصال نفسه.

## ماذا يحدث داخليًا؟

`PrismaPg` يمسك مشغّل `pg` ومجمعه. استدعاءات `prisma.book.findMany` لاحقًا تستعير اتصالًا من المجمع وتعيده. `$disconnect` ينهي المجمع حتى لا تبقى مقابس بعد الاختبار أو بعد إشارة الإيقاف.

من غير `enableShutdownHooks`، إيقاف العملية بإشارة قد لا يمر على `onModuleDestroy`. استدعاء `app.close()` في الاختبار يفعل ذلك.

لا تنشئ `PrismaClient` ثانيًا بجانب الخدمة. نسختان تعنيان مجمعين على القاعدة نفسها بلا داعٍ.

## مثال عملي

اختبار e2e يستدعي `app.close()` في `afterAll` فيمر `$disconnect`. اختبار وحدة لخدمة الكتب يستبدل `PrismaService` بقيمة مزيفة ولا يفتح PostgreSQL. إن بقي `onModuleInit` على النسخة الحقيقية داخل اختبار وحدة، كل اختبار يدفع ثمن اتصال.

عنوان التطوير يشير إلى قاعدة `library`. عنوان الاختبار يشير إلى `library_test`. المخطط الذي رفض العنوان الفارغ يبقى كما هو.

## أخطاء شائعة

- `new PrismaClient()` بلا `adapter` على Prisma 7.
- نسيان `.js` في استيراد العميل المولَّد تحت ESM.
- فتح عميل في المتحكم.
- الاعتماد على الاتصال الكسول ثم مفاجأة أول طلب بخطأ شبكة.

## متى نستخدمه؟

عند إقلاع أي تطبيق دورة يستخدم Prisma على PostgreSQL.

## متى لا نستخدمه؟

داخل سكربت `prisma` الذي يدير CLI. ذلك يقرأ التهيئة الخاصة به. التطبيق يستخدم الخدمة.

## مقارنة

| عميل في كل خدمة | `PrismaService` واحد |
| --- | --- |
| مجمعات متعددة | مجمع واحد |
| إغلاق غير واضح | `$disconnect` في الخطاف |
| عنوان من `process.env` المبكر | عنوان من `ConfigService` |

## تمارين

### تمرين سهل

في أي خطاف يُستدعى `$connect`؟

<details>
<summary>الحل</summary>

`onModuleInit`.

</details>

### تمرين متوسط

لماذا المحوّل `PrismaPg` وليس سلسلة العنوان وحدها في `super` على Prisma 7؟

<details>
<summary>الحل</summary>

Prisma ORM 7 في توثيق Nest يبني العميل بمحوّل مشغّل، مثل `PrismaPg`، أو بعنوان Accelerate. صفحة Prisma في Nest تمرّر `connectionString` إلى المحوّل.

</details>

### تمرين متقدم

اختبارات e2e تترك اتصالات مفتوحة حتى تمتلئ قاعدة الاختبار. ماذا تضيف في `main` وفي نهاية الاختبار؟

<details>
<summary>الحل</summary>

`app.enableShutdownHooks()` حتى تُستدعى خطافات الإيقاف مع الإشارة، و `await app.close()` في نهاية الاختبار حتى يعمل `onModuleDestroy` ويستدعي `$disconnect`.

</details>

## أسئلة مراجعة

- من أين يُقرأ `DATABASE_URL` داخل الخدمة؟
- ماذا يغلق `$disconnect`؟
- لماذا تفشل الإقلاع أفضل من فشل أول طلب؟

## الخلاصة

الاتصال مزوّد واحد يمتد العميل، يبني `PrismaPg` من الإعداد، ويتصل وينقطع مع حياة الوحدة.

## ماذا بعد؟

[Prisma: Schema](./05-prisma-schema.md) يصف جدولي الكتب والأعضاء في `schema.prisma`.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
