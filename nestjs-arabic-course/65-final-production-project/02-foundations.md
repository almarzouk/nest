# الأساسات: الإعداد وقاعدة البيانات

## الهدف من هذا الدرس

إقلاع هيكل رف: تطبيق ESM، إعداد بيئة، وPrisma يتصل بـ Postgres، مع مسار صحة لا يلمس جداول الكتب بعد.

## المشكلة التي يحلها

الدرس الأول يكتب كل الموارد في `AppModule`. سر القاعدة في المستودع. الصحة تفحص Stripe فتسقط الحاوية عندما يتأخر المزود. المشروع CommonJS بينما بقية المقتطفات تستورد بلاحقة `.js`.

## الفكرة ببساطة

`nest new` بـ ESM (الافتراضي في Nest 12 ومعه Vitest لاحقًا). `ConfigModule` يقرأ `DATABASE_URL`. `PrismaService` singleton يتصل وينقطع مع دورة الحياة. `GET /health` يفحص العملية وربما القاعدة، لا الدفع. لا جداول مشاريع في هذا الدرس غير ما يلزم للإقلاع إن أخّرتم المخطط إلى درس المنظمة.

## التعريف التقني

ولّدوا التطبيق بلا فرض Observe إن كنتم تتدربون: `nest new raf-api --no-observe`. التفعيل لاحقًا `--observe` مع `instrument`. Express الافتراضي. `PrismaClient` واحد في مزوّد `onModuleInit` / `onModuleDestroy`. لا `new PrismaClient()` داخل الطلب. متغيرات بلا قيم افتراضية للأسرار. الصحة عبر Terminus كما في وصفات Nest، وفحص Postgres منفصل عن فحص «حي» بسيط حتى يستطيع الموازن التفريق.

## Mental Model

```text
main.ts → AppModule
            ConfigModule
            PrismaModule
            HealthModule
لا BooksModule بعد
```

## أبسط مثال

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

## شرح المثال

- المزود singleton. نطاق REQUEST هنا يعيد الاتصال بلا فائدة في نموذج pool.
- الاستيراد من `@prisma/client` بلا مسار `.js` لأن الحزمة تحدده. ملفاتكم أنتم تستخدم `.js` في الاستيراد المحلي.
- الفشل في `$connect` يوقف الإقلاع وهذا أفضل من أول طلب غامض.

## ماذا يحدث داخليًا؟

1. `ConfigModule` يفشل الإقلاع إن غاب `DATABASE_URL` عندما تضبطون `validate`.
2. Prisma يفتح مجمعًا.
3. `GET /health/live` يعيد 200 بلا قاعدة.
4. `GET /health/ready` يفحص الاتصال.
5. لا مسار مشروع بعد. جدول `Project` يأتي مع درس المشاريع حتى لا يُهاجر مخطط بلا مستأجر.

## مثال عملي

```ts
import { z } from 'zod';

export const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
});
```

`@nestjs/config` في Nest 12 يقبل Standard Schema. لا تضعوا `STRIPE_SECRET_KEY` كحقل إجباري في هذا الدرس إن كان الدفع مرحلة 06؛ أضيفوه عندما تصل الوحدة حتى يقلع التمرين المحلي بلا مفتاح.

## أخطاء شائعة

- سر في `docker-compose` يُcommit مع كلمة مرور إنتاج.
- صحة واحدة تفحص Stripe وPrisma معًا.
- ESM بلا `"type": "module"` ثم فشل الاستيراد.
- فتح عميل Prisma لكل وحدة ميزة.

## متى نستخدمه؟

قبل أي مورد. الهيكل الذي لا يقلع لا يستقبل منظمة.

## متى لا نستخدمه؟

لا تضيفوا Redis في هذا الدرس إلا كخدمة فارغة في Compose لاحقًا. المهمة في درس 09. ولا تختاروا Fastify هنا بلا قياس.

## مقارنة

| المسار | يفحص |
| --- | --- |
| `/health/live` | العملية |
| `/health/ready` | Postgres |
| النشر | ليس بعد |

## تمارين

### تمرين سهل

لماذا عميل Prisma واحد؟

<details>
<summary>الحل</summary>

المجمع يُدار مرة. عميل لكل طلب يستنفد اتصالات Postgres.

</details>

### تمرين متوسط

ماذا يحدث لموازن إن كانت الصحة الوحيدة تنادي Stripe؟

<details>
<summary>الحل</summary>

تعطل المزود يسحب نسخ رف السليمة. افصلوا الحي عن الجاهزية، واتركوا Stripe خارج الاثنين.

</details>

### تمرين متقدم

اكتبوا متغيرين إجباريين اليوم ومتغيرًا يُضاف في درس الدفع.

<details>
<summary>الحل</summary>

اليوم `DATABASE_URL`. `PORT` له افتراض. في درس الدفع يُضاف `STRIPE_SECRET_KEY` وسر الويب هوك إلى المخطط حتى لا يقلع الإنتاج بدونهما، بينما التطوير بلا فوترة يمكنه وحدة لا تُستورد.

</details>

## أسئلة مراجعة

- أي مشغّل اختبار سياتي مع ESM؟
- ماذا يفعل live مقابل ready؟
- لماذا لا جدول المشاريع هنا؟
- أين لا يُوضع سر القاعدة؟

## الخلاصة

رف تقلع بـ ESM وإعداد متحقق وPrisma واحد وصحة لا تعرف الدفع. الموارد تبدأ بعد هذا الهيكل.

## ماذا بعد؟

أول سلوك للمستخدم: إثبات الهوية. الدرس التالي Auth.

## المصادر الرسمية

- NestJS — Configuration: https://docs.nestjs.com/application/configuration
- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- NestJS — Terminus: https://docs.nestjs.com/recipes/terminus
