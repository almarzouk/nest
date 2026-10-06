# Caching كقرار أداء

## الهدف من هذا الدرس

اختيار ما يُعاد استخدامه بين طلبات متجر رف، ومفتاح التخزين، ومدة الصلاحية، ولحظة الإبطال، باستخدام أدوات Nest للتخزين المؤقت لا كطبقة تُلف كل `GET`.

## المشكلة التي يحلها

غلاف الكتاب وصفحة التصنيف تُقرأ أكثر بكثير مما تُكتب. كل قراءة تعيد نفس `findMany`. بعد الفهرس، الزمن مقبول لكن القاعدة ما زالت تدفع الثمن في كل طلب. التخزين يقلل هذا التكرار إلى أن يتغير الكتالوج.

## الفكرة ببساطة

التخزين المؤقت نسخة سريعة لناتج غالٍ، لها مفتاح ومدة. الطلب التالي بنفس المفتاح يقرأ النسخة. أي كتابة تغيّر المعنى تحذف المفتاح أو تنتظر انتهاء المدة. تخزين بلا إبطال يعرض سعرًا قديمًا.

## التعريف التقني

`CacheModule` من `@nestjs/cache-manager` يسجّل مخزنًا. `CacheInterceptor` مع `@CacheKey` و `@CacheTTL` يخزّن ردود `GET` العامة. المفتاح الافتراضي مرتبط بالمسار؛ إن اختلف الرد حسب المستأجر أو المستخدم وجب أن يدخل ذلك في المفتاح. للخدمة، احقن `CACHE_MANAGER` واقرأ و اكتب بنفسك عندما لا يكون الرد HTTP كله قابلًا للتخزين. مدة `ttl` وحدتها في صفحة Caching الحالية للحزمة التي ولّدها مشروعك؛ لا تنقل رقمًا من مقال قديم. المخزن داخل الذاكرة لا يُشارك بين عمليات، وRedis يشاركها.

## Mental Model

```text
مفتاح: tenantId + "published-books"
قراءة: إصابة → رد النسخة
        غياب → استعلام → خزّن مع TTL
كتابة كتاب: احذف المفتاح ثم احفظ
```

## أبسط مثال

```ts
import { CacheInterceptor, CacheTTL } from '@nestjs/cache-manager';
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { BooksService } from './books.service.js';

@Controller('books')
@UseInterceptors(CacheInterceptor)
export class BooksController {
  constructor(private readonly books: BooksService) {}

  @Get('featured')
  @CacheTTL(30_000)
  featured() {
    return this.books.featured();
  }
}
```

## شرح المثال

- `featured` قائمة عامة واحدة. المفتاح من المسار يكفي.
- ثلاثون ثانية تعني أن ظهور كتاب مميز قد يتأخر حتى هذه المدة، وهذا عقد منتج لا تفصيل إطار.
- المسار الذي يعتمد على `Authorization` لا يُلف بهذا المعترض كما هو، وإلا شارك الزوار نفس الجسم.

## ماذا يحدث داخليًا؟

1. المعترض يبني مفتاحًا قبل تنفيذ الـ Handler.
2. الإصابة تعيد القيمة ولا تستدعي الخدمة.
3. الغياب يشغّل الـ Handler ثم يخزن القيمة القابلة للتسلسل.
4. انتهاء `ttl` يحذف النسخة فيقرأ الطلب التالي المصدر.
5. الحذف الصريح عند `update` أقصر من انتظار المدة حين يكون الخطأ في السعر مكلفًا.

## مثال عملي

```ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject, Injectable } from '@nestjs/common';
import type { Cache } from 'cache-manager';
import { PrismaService } from '../prisma/prisma.service.js';

@Injectable()
export class BooksService {
  constructor(
    private readonly prisma: PrismaService,
    @Inject(CACHE_MANAGER) private readonly cache: Cache,
  ) {}

  async listPublished(tenantId: string) {
    const key = `books:${tenantId}:published`;
    const hit = await this.cache.get(key);
    if (hit) return hit;
    const rows = await this.prisma.book.findMany({
      where: { tenantId, status: 'published' },
      take: 20,
      orderBy: { publishedAt: 'desc' },
    });
    await this.cache.set(key, rows, 30_000);
    return rows;
  }

  async publish(tenantId: string, id: string) {
    const book = await this.prisma.book.update({
      where: { id },
      data: { status: 'published' },
    });
    await this.cache.del(`books:${tenantId}:published`);
    return book;
  }
}
```

المفتاح يتضمن `tenantId`. الحذف بعد النشر يمنع بقاء القائمة القديمة حتى نهاية المدة. تحقق أن `id` يخص المستأجر قبل `update`؛ التخزين لا يعوّض عزل البيانات.

## أخطاء شائعة

- تخزين رد فيه سعر المستخدم أو سلة خاصة بمفتاح المسار فقط.
- نسيان الحذف بعد الكتابة والاعتماد على TTL طويل لبيانات تتغير فورًا.
- تخزين كائن فيه دوال أو تواريخ غير متسلسلة ثم فشل صامت عند القراءة من Redis.
- وضع التخزين أمام مسار أبطأ بسبب N+1. أصلح الاستعلام أولًا حتى لا تخزن الناتج الغالي الخطأ.

## متى نستخدمه؟

قراءات كثيرة، ناتج مستقر لثوانٍ أو دقائق، ومفتاح يساوي كل ما يغيّر الجسم: مستأجر، لغة، معاملات الصفحة.

## متى لا نستخدمه؟

أوامر الدفع، الرصيد، وأي `GET` يخص شخصًا واحدًا إن لم يكن المفتاح هويته وكنت مستعدًا لإبطاله عند تغيّر حسابه. ولا كبديل لفهرس مفقود على مسار يُستدعى مرة في الدقيقة.

## مقارنة

| المكان | يشارك بين العمليات؟ | يناسب |
| --- | --- | --- |
| ذاكرة العملية | لا | كتالوج تجربة، عملية واحدة |
| Redis عبر `cache-manager` | نعم | أكثر من نسخة من API |
| CDN أمام GET العام | نعم، خارج Nest | أغلفة وملفات لا تخص مستأجرًا |

## تمارين

### تمرين سهل

`GET /books/featured` عام. هل يكفي مفتاح المسار؟

<details>
<summary>الحل</summary>

نعم إن كان الجسم واحدًا لكل العالم. إن صار لكل مستأجر أبطال مختلفون، أضف `tenantId` إلى المفتاح وأوقف المعترض العام أو استبدله بتخزين داخل الخدمة.

</details>

### تمرين متوسط

نشر كتاب لا يظهر في القائمة إلا بعد نصف دقيقة. أين الخلل إن كان TTL هو 30 ثانية والحذف غير موجود؟

<details>
<summary>الحل</summary>

النسخة القديمة تبقى حتى انتهاء المدة. `publish` يجب أن يستدعي `cache.del` لمفتاح تلك القائمة، أو أن تكون نصف الدقيقة مقبولة في عقد المنتج ومكتوبة للفريق.

</details>

### تمرين متقدم

عمليتان خلف موازن حمل، والمخزن `memory`. لماذا يرى عميل قائمة قديمة رغم `del` في العملية التي نفذت النشر؟

<details>
<summary>الحل</summary>

كل عملية لها ذاكرتها. `del` يمسح عملية واحدة. الموازن قد يرسل القراءة التالية إلى العملية الأخرى. انقل المخزن إلى Redis مشترك، أو اقبل بالتباين حتى نهاية TTL على كل عملية.

</details>

## أسئلة مراجعة

- ماذا يجب أن يدخل في المفتاح غير المسار؟
- متى يكون TTL كافيًا، ومتى يلزم `del`؟
- لماذا التخزين لا يسبق إصلاح الاستعلام؟
- ما فرق ذاكرة العملية عن Redis هنا؟

## الخلاصة

التخزين المؤقت عقد على freshness: مفتاح كامل، مدة معروفة، وإبطال عند الكتابة التي لا تحتمل الانتظار. `CacheModule` ينفذ العقد. هو لا يختار ما يصح تخزينه.

## ماذا بعد؟

الجسم المخزّن قد يبقى كبيرًا على السلك. الدرس التالي عن ضغط الاستجابة، ومتى يُنقل هذا العمل إلى الوكيل.

## المصادر الرسمية

- NestJS — Caching: https://docs.nestjs.com/data/caching
