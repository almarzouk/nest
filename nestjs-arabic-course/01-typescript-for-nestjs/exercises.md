# تمارين الفصل

## تمرين سهل

اكتب نوع دالة `isExpensive(price: number): boolean` تعيد `true` عندما `price > 100`. ما نوع الناتج الذي يستنتجه المترجم لـ `const flag = isExpensive(40)`؟

<details>
<summary>الحل</summary>

```ts
function isExpensive(price: number): boolean {
  return price > 100;
}
const flag = isExpensive(40);
```

نوع `flag` هو `boolean`.

</details>

## تمرين متوسط

صنف `ReviewsController` يحتاج `ReviewsService` في حقل خاص غير قابل لإعادة التعيين. اكتب الـ constructor المختصر ودالة `async count()` ترجع `Promise<number>` عبر `this.reviewsService.count()`، مع افتراض أن الخدمة فيها `count(): Promise<number>`.

<details>
<summary>الحل</summary>

```ts
class ReviewsController {
  constructor(private readonly reviewsService: ReviewsService) {}

  async count(): Promise<number> {
    return this.reviewsService.count();
  }
}
```

</details>

## تمرين متقدم

ملف فيه الواجهة `AppConfig` والصنف `ConfigService`، والمتحكم يستورد الاثنين بـ `import type` ويحقن `ConfigService`. ماذا ينكسر وقت تشغيل Nest حتى لو نجح عرض الأنواع في المحرر، وما الاستيراد الصحيح؟

<details>
<summary>الحل</summary>

`import type` يمحو `ConfigService` من JavaScript، فـ `design:paramtypes` لا يحصل على الدالة البناء، والحاوية لا تجد الرمز. الواجهة أصلًا لا تُحقن. الصحيح:

```ts
import { ConfigService } from './config.service.js';
```

وأبقِ `import type` للواجهة فقط إن استخدمتها كنوع ممحو. الخدمة التي في constructor تُستورد كقيمة، ويبقى `emitDecoratorMetadata` وزينة الصنف حتى تُصدر الأنواع.

</details>
