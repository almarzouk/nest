# أمثلة عملية

مساران بلا حارس. الاستيراد ينتهي بـ `.js`. الحياة لا ترى Prisma. الجاهزية تجمع القاعدة والكاش.

## الحياة

```ts
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class LiveController {
  @Get('live')
  live(): { status: string } {
    return { status: 'ok' };
  }
}
```

## وحدة Terminus

```ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HealthController } from './health.controller.js';
import { RedisHealthIndicator } from './redis.health.js';

@Module({
  imports: [
    TerminusModule.forRoot({ gracefulShutdownTimeoutMs: 1000 }),
  ],
  controllers: [HealthController],
  providers: [RedisHealthIndicator],
})
export class HealthModule {}
```

`enableShutdownHooks()` في `main.ts` حتى يصير الرد `shutting_down`.

## الجاهزية: قاعدة لازمة وكاش متدهور

```ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  HealthIndicatorService,
  PrismaHealthIndicator,
} from '@nestjs/terminus';
import { PrismaService } from '../prisma/prisma.service.js';
import type { Redis } from 'ioredis';

@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly indicators: HealthIndicatorService,
    private readonly prismaHealth: PrismaHealthIndicator,
    private readonly prisma: PrismaService,
    private readonly redis: Redis,
  ) {}

  @Get('ready')
  @HealthCheck()
  ready() {
    return this.health.check([
      () => this.prismaHealth.pingCheck('database', this.prisma).withTimeout(1000),
      () => this.cache(),
    ]);
  }

  private async cache() {
    const indicator = this.indicators.check('cache');
    try {
      await this.redis.ping();
      return indicator.up();
    } catch {
      return indicator.degraded('cache unreachable, serving without cache');
    }
  }
}
```

`Redis` يُقدَّم كمزود `useFactory` في وحدة البيانات، لا يُنشأ داخل المتحكم. إن كان الطابور لا يعمل بلا Redis فاستبدل `degraded` بـ `attempt(() => this.redis.ping())` حتى يصبح الفشل `down`.

## ماذا تتأكد منه

1. `/health/live` بلا `HealthCheckService`.
2. مهلة المؤشر أقصر من مهلة المسبار.
3. لا `AuthGuard` ولا كاش Nginx على المسارين.
4. TypeORM، إن استُعمل، `pingCheck('database').withTimeout(1000)` وينفّذ `SELECT 1`.
5. لا تبحث عن مؤشر Redis جاهز في الحزمة.
