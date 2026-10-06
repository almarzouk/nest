# @nestjs/terminus

## الهدف من هذا الدرس

تركيب `@nestjs/terminus` بحيث يجمع مؤشرات الجاهزية في `HealthCheckResult`، ويرد `503` عند العطل، و `shutting_down` عند الإيقاف.

## المشكلة التي يحلها

كتابة `try/catch` حول `SELECT 1` في كل مشروع تعيد اختراع شكل الرد والمهلة والرمز. Terminus يعرّف `status` و `info` و `error` و `details`، وينفّذ المحاولات، ويحوّل الفشل إلى حالة صحية لا إلى استثناء `500` إن استعملت الواجهة المخصصة لذلك.

## الفكرة ببساطة

`TerminusModule` في وحدة الصحة. متحكم عليه `@HealthCheck()` يستدعي `HealthCheckService.check` بمصفوفة دوال. كل دالة مؤشر. إن نجح الجميع فـ `status` هو `ok` و HTTP `200`. إن نزل واحد فـ `error` و `503`. مؤشر `degraded` لا يفشل الفحص ويُبقي `200` مع حالة عامة `degraded`.

## التعريف التقني

```bash
npm install --save @nestjs/terminus
```

`HttpHealthIndicator` يحتاج أيضًا `@nestjs/axios` و `axios` إن فحصت عنوانًا خارجيًا. قاعدة البيانات و الذاكرة والقرص لا تحتاجه.

```ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HealthController } from './health.controller.js';

@Module({
  imports: [
    TerminusModule.forRoot({
      gracefulShutdownTimeoutMs: 1000,
    }),
  ],
  controllers: [HealthController],
})
export class HealthModule {}
```

```ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService } from '@nestjs/terminus';
import { PrismaHealthIndicator } from '@nestjs/terminus';
import { PrismaService } from '../prisma/prisma.service.js';

@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly prisma: PrismaHealthIndicator,
    private readonly db: PrismaService,
  ) {}

  @Get('ready')
  @HealthCheck()
  ready() {
    return this.health.check([
      () => this.prisma.pingCheck('database', this.db).withTimeout(1000),
    ]);
  }
}
```

`PrismaHealthIndicator.pingCheck` ينفّذ `SELECT 1` عبر `$queryRawUnsafe`، أو أمر ping إن كان العميل Mongo. المهلة الافتراضية نحو 1000 مللي ثانية، و `.withTimeout()` أوضح من خيار `timeout` القديم الذي صار مهملًا. `cacheFor(5000)` يمنع ضرب القاعدة في كل مسبار إن كانت الفترة أقصر من اللازم.

مع `enableShutdownHooks()`، بعد `SIGTERM` يصير `status` القيمة `shutting_down` و HTTP `503`. `gracefulShutdownTimeoutMs` يؤخر الإغلاق على `SIGTERM` فقط حتى يسحب المنسّق المرور. `SIGINT` يعلّم الحالة ثم يغلق فورًا. بلا خطافات الإيقاف لا يرى Terminus الإشارة.

رمي خطأ من مؤشر مبني بـ `up()` و `down()` يُعد عيبًا ويعيد `500`. `attempt()` يحوّل الرمي إلى `down`.

## Mental Model

`health.check` يعيد `ok` مع 200، أو `error` مع 503، أو `degraded` مع 200، أو `shutting_down` مع 503 بعد `SIGTERM`.

## أبسط مثال

المتحكم أعلاه على `GET /health/ready`. `pingCheck` يعيد محاولة، و `withTimeout(1000)` يحدها، و `check` يبني `HealthCheckResult`. الحياة لا تمر من `check`.

## شرح المثال

المفتاح `'database'` يظهر تحت `info` عند الصعود وتحت `error` عند النزول. الحياة تبقى الدالة الرخيصة من الدرس السابق.

## ماذا يحدث داخليًا؟

1. المزخرف يمرر النتيجة إلى مرشح Terminus.
2. المؤشر الناجح يذهب إلى `info` و `details`.
3. المؤشر النازل يذهب إلى `error` و `details` ويقلب `status` إلى `error`.
4. المهلة تُجهض الإشارة الممررة إلى العملية.
5. `cacheFor` يعيد النتيجة نفسها ويضع `cachedResponse: true`.
6. عند الإغلاق تُستبدل الحالة العامة بـ `shutting_down` قبل قطع الاتصالات.

## مثال عملي

جاهزية المتجر: Prisma بمهلة ثانية وكاش خمس ثوانٍ. قرص `checkStorage('storage', { path: '/', thresholdPercent: 0.9 })` على مسار تشخيص لا على الحياة. الذاكرة `checkHeap` تقارن `heapUsed` بالبايت. لا تضع عتبة منخفضة على مسبار يقتل الحاوية.

## أخطاء شائعة

- فحص وثائق Nest عبر HTTP داخل كل جاهزية.
- خيار `timeout` القديم بدل `withTimeout()`.
- نسيان `enableShutdownHooks()` فتبقى الجاهزية `ok` أثناء القتل.
- `gracefulShutdownTimeoutMs` مع توقع التأخير على Ctrl+C.
- حارس على المتحكم.

## متى نستخدمه؟

عندما يسأل منسق عن الجاهزية بشكل موحّد.

## متى لا نستخدمه؟

لا تلف كل مسار عمل بـ `@HealthCheck()`. هو لمسبار البنية. ولا تستعمل مؤشر HTTP على خدمتك نفسها فتصنع حلقة.

## مقارنة

| الحالة | HTTP | المعنى |
| --- | --- | --- |
| `ok` | 200 | المؤشرات صاعدة |
| `degraded` | 200 | تدهور لا يوقف الخدمة |
| `error` | 503 | مؤشر نازل |
| `shutting_down` | 503 | الإشارة وصلت |

## تمارين

### تمرين سهل

أي مزخرف يضع نتيجة `health.check` في رد الصحة؟

<details>
<summary>الحل</summary>

`@HealthCheck()` على دالة المتحكم.

</details>

### تمرين متوسط

لماذا `gracefulShutdownTimeoutMs` لا يؤخر Ctrl+C؟

<details>
<summary>الحل</summary>

التأخير على `SIGTERM` الذي يرسله المنسّق. `SIGINT` يعلّم `shutting_down` ثم يغلق فورًا. الخطافات يجب أن تكون مفعّلة في الحالتين.

</details>

### تمرين متقدم

مؤشر يرمي `Error` من دالة تستدعي `down()` بعد `throw`. أي رمز يرى المنسّق؟

<details>
<summary>الحل</summary>

`500` لأن الرمي خارج `attempt()` عيب لا حالة `down`. لف العملية في `attempt()` ليصبح `503` مع `status: error`.

</details>

## أسئلة مراجعة

- ماذا يثبت الحزمة قبل المؤشرات؟
- أين تُضبط مهلة الإغلاق اللطيف؟
- ماذا يفعل `cacheFor`؟

## الخلاصة

Terminus يجمع محاولات المؤشرات في رد واحد: `200` للصعود والتدهور، و `503` للنزول وللإغلاق. فعّل خطافات الإيقاف، وحدّ المهلة بـ `withTimeout`، وأبقِ الحياة خارج `check`.

## ماذا بعد؟

مؤشرات القاعدة و Redis: ماذا يفشل الجاهزية وماذا يبقى متدهورًا.

## المصادر الرسمية

- NestJS — Terminus: https://docs.nestjs.com/recipes/terminus
