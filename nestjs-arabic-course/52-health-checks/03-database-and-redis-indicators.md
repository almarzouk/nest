# مؤشرات قاعدة البيانات و Redis

## الهدف من هذا الدرس

اختيار المؤشر الذي يطابق التبعية: TypeORM أو Prisma للقاعدة، ومؤشر تبنيه لـ Redis، مع قرار `down` أو `degraded`.

## المشكلة التي يحلها

`@nestjs/terminus` يأتي بمؤشرات قاعدة و HTTP و gRPC وذاكرة وقرص. لا يوجد `RedisHealthIndicator` جاهز. نسخ فحص HTTP على Redis أو اعتبار الكاش عطلًا قاتلًا يُخرج النسخ من الخدمة كلما تأخر المخبأ. طابور لا يُنشر بدونه يجب أن يُنزل الجاهزية فعلًا.

## الفكرة ببساطة

القاعدة التي تخدم القراءة والكتابة: `down` عند فشل `SELECT 1`. كاش يمكن الخدمة بدونه: `degraded` فيبقى HTTP `200`. Redis الطابور: `down`. المهلة قصيرة حتى لا يعلق المسبار. المفتاح في الرد اسم واضح مثل `database` أو `cache`.

## التعريف التقني

TypeORM، بعد أن يكون الاتصال قائمًا، ينفّذ `SELECT 1` (أو ما يعادله على Oracle و SAP HANA):

```ts
() => this.db.pingCheck('database').withTimeout(1000)
```

اتصال مسمّى يُمرَّر في الخيار `connection` مع `DataSource` المحقون بـ `@InjectDataSource('albumsConnection')`.

Prisma:

```ts
() => this.prismaHealth.pingCheck('database', this.prisma).withTimeout(1000)
```

العميل يُحقن كما حقنته في التطبيق. المؤشر يستدعي أمر ping لـ Mongo أو `$queryRawUnsafe('SELECT 1')` لغيرها.

Redis بمؤشر خاص. الحزمة لا تصدّر واحدًا:

```ts
import { Injectable } from '@nestjs/common';
import { HealthIndicatorService } from '@nestjs/terminus';
import type { Redis } from 'ioredis';

@Injectable()
export class RedisHealthIndicator {
  constructor(
    private readonly health: HealthIndicatorService,
    private readonly redis: Redis,
  ) {}

  cache(key: string) {
    return this.health.check(key).attempt(async () => {
      await this.redis.ping();
    }).withTimeout(1000);
  }
}
```

للكاش الذي تتحمل غيابه استعمل `indicator.degraded('cache unreachable')` بعد فشل `ping`، لا `down`. `degraded` يظهر تحت `info`، والحالة العامة `degraded`، و HTTP يبقى `200`. أي مؤشر `down` في نفس الفحص يغلب ذلك ويصبح `error`. طابور لا يعمل بلا Redis يستعمل `attempt` حتى يُسجَّل `down`.

الذاكرة: `checkHeap('memory_heap', 150 * 1024 * 1024)` يقارن `heapUsed`. `checkRSS` يقارن الذاكرة المقيمة. القرص: `checkStorage('storage', { path: '/', thresholdPercent: 0.5 })` أو `threshold` بالبايت. هذه مؤشرات تشخيص. لا تلصقها بمسار الحياة.

## Mental Model

```text
Postgres لازم     -> pingCheck + down
Redis كاش         -> degraded و 200
Redis طابور لازم  -> down و 503
لا مؤشر Redis جاهز في الحزمة
```

## أبسط مثال

جاهزية المتجر: `pingCheck` على Prisma فقط. الكاش في دالة ثانية على مسار `/health/ready` إن قررت أن البطء مقبول، باستعمال `degraded`.

## شرح المثال

`attempt` يعلّم `up` إن انتهى `ping`، و `down` إن رمى أو انتهت المهلة. `degraded` قرارك أنت عندما النجاح ليس ثنائيًا. لا تغلّف `degraded` ثم ترمي بعده.

## ماذا يحدث داخليًا؟

1. TypeORM أو Prisma يرسل استعلامًا رخيصًا على الاتصال القائم.
2. المهلة تُجهض المحاولة وتكتب `timeout of 1000ms exceeded`.
3. Redis `PING` على عميل ioredis الذي فتحته عند الإقلاع.
4. `degraded` لا يملأ كائن `error`.
5. `down` يملأه ويقلب الحالة إلى `error`.
6. المنسّق يرى `503` فيسحب النسخة، أو `200` فيبقيها مع تنبيه على `degraded`.

## مثال عملي

نسختان، Postgres لازم، Redis للكتب الشائعة فقط. الجاهزية: مؤشر القاعدة `down` عند الفشل، ومؤشر الكاش `degraded`. انقطاع Redis يبقي `200` وتزداد القراءة على القاعدة. إن كان Redis طابور الفواتير الوحيد، بدّل المؤشر إلى `down` حتى لا تقبل النسخة طلبات لن تُنشر.

## أخطاء شائعة

- البحث عن `RedisHealthIndicator` في الحزمة.
- `degraded` لطابور لا مسار بديل له.
- `SELECT 1` على الحياة.
- عتبة قرص 50٪ على مسبار يقتل الحاوية.
- مهلة أطول من مهلة المسبار فيعلّق المنسّق قبل Terminus.

## متى نستخدمه؟

في `/health/ready` لكل تبعية تغيّر قرار التوزيع.

## متى لا نستخدمه؟

لا تفحص كل مفتاح Redis ولا تشغّل استعلامًا ثقيلًا. `SELECT 1` و `PING` يكفي. لا تضع المؤشر على كل طلب مستخدم.

## مقارنة

| التبعية | الأداة | عند الفشل |
| --- | --- | --- |
| TypeORM | `pingCheck().withTimeout` | `down` |
| Prisma | `PrismaHealthIndicator` | `down` |
| Redis كاش | مؤشر خاص + `degraded` | تبقى النسخة |
| Redis طابور | مؤشر خاص + `attempt` | `503` |
| ذاكرة / قرص | `checkHeap` / `checkStorage` | تشخيص لا حياة |

## تمارين

### تمرين سهل

ماذا ينفّذ `TypeOrmHealthIndicator` على Postgres؟

<details>
<summary>الحل</summary>

`SELECT 1`.

</details>

### تمرين متوسط

لماذا الكاش `degraded` لا `down` في المتجر؟

<details>
<summary>الحل</summary>

لأن الصفحات تُقرأ من القاعدة عند غياب المخبأ. `degraded` يُبقي HTTP `200` ويظهر في المراقبة. `down` كان سيخرج النسخ كلها.

</details>

### تمرين متقدم

مهلة المؤشر 3000 مللي ثانية ومسبار Kubernetes ينتظر 1000. ماذا يرى المنسّق عند تعليق القاعدة؟

<details>
<summary>الحل</summary>

انتهاء مهلة المسبار قبل رد Terminus، فيُحسب الفشل غامضًا وقد يُعاد التشغيل. اجعل `withTimeout` أقصر من مهلة المسبار.

</details>

## أسئلة مراجعة

- هل تصدر الحزمة مؤشر Redis؟
- أين يعيش `degraded` في JSON؟
- ماذا يقارن `checkHeap`؟

## الخلاصة

القاعدة اللازمة تُسقِط الجاهزية عبر `pingCheck`. Redis يُبنى بمؤشر خاص: `degraded` للكاش و `down` للطابور الذي لا بديل له. الذاكرة والقرص تشخيص، والحياة تبقى بلا هذه الاستعلامات.

## ماذا بعد؟

الأداء: أين يضيع الوقت بعد أن صارت النسخة تعرف إن كانت جاهزة.

## المصادر الرسمية

- NestJS — Terminus: https://docs.nestjs.com/recipes/terminus
- NestJS — Redis: https://docs.nestjs.com/microservices/redis
