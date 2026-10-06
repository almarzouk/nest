# أمثلة عملية

هذا الملف يمر على متجر رف تحت قياس واحد. كل مثال يقرر طبقة مختلفة: الحلقة، نطاق الحقن، الفهرس، الكاش، الضغط، العمليات، ثم المحول. لا تُلصق هذه الطبقات دفعة واحدة في `main.ts` قبل أن تعرف أي طبقة ظهر رقمها في القياس.

## المثال 1 — الصحة تكشف السد

```ts
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class HealthController {
  @Get()
  live() {
    return { ok: true, at: Date.now() };
  }
}
```

أثناء توليد تقرير أسعار متزامن، راقب `at` بين طلبين متتاليين على `/health`. الفجوة هي زمن السد على هذه العملية. إن بقيت الفجوة صغيرة وتأخر `GET /books` وحده، فالسقف تحت الخدمة لا في الحلقة.

## المثال 2 — فهرس القائمة ثم كاش بمفتاح المستأجر

```ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject, Injectable } from '@nestjs/common';
import type { Cache } from 'cache-manager';
import { PrismaService } from '../prisma/prisma.service.js';

@Injectable()
export class ShelfService {
  constructor(
    private readonly prisma: PrismaService,
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
  ) {}

  async published(tenantId: string) {
    const key = `shelf:${tenantId}`;
    const hit = await this.cache.get(key);
    if (hit) return hit;
    const rows = await this.prisma.book.findMany({
      where: { tenantId, status: 'published' },
      orderBy: { publishedAt: 'desc' },
      take: 20,
      select: { id: true, title: true, publishedAt: true },
    });
    await this.cache.set(key, rows, 15_000);
    return rows;
  }
}
```

الفهرس `(tenantId, status, publishedAt DESC)` يجعل الغياب رخيصًا. الكاش يجعل التكرار أرخص. المفتاح فيه المستأجر حتى لا تعرض مكتبة قائمة مكتبة أخرى. عند النشر احذف `shelf:${tenantId}` لا كل المفاتيح.

## المثال 3 — عمال للـ CPU، Express يبقى حتى القياس

عملية الويب تبقى على Express. تقرير المدرج عملية أخرى:

```ts
import cluster from 'node:cluster';
import { availableParallelism } from 'node:os';

export function startWeb(boot: () => Promise<void>) {
  if (!cluster.isPrimary) return boot();
  for (let i = 0; i < availableParallelism(); i++) cluster.fork();
}
```

لا تستدعِ `startWeb` إلا بعد أن يُظهر القياس أن نواة واحدة ممتلئة والقاعدة مرتاحة. الجلسة والكاش في الذاكرة يُنقلان قبل هذه الدالة. Fastify لا يدخل هذا الملف: صفحة الأداء تطلب دليلًا، وعنوان الاستماع عندها `'0.0.0.0'`.

## ماذا تتأكد منه

- `/health` لا يلمس Prisma، وإلا قاس القاعدة وسمّاها حلقة.
- لا `Scope.REQUEST` على مسار القائمة إن كان المستأجر يُمرَّر وسيطًا إلى singleton.
- `take` داخل الاستعلام، والضغط في الوكيل إن كان Nginx موجودًا.
