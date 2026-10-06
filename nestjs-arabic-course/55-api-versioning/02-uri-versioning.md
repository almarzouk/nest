# URI Versioning

## الهدف من هذا الدرس

تشغيل نسختي كتالوج رف على المسار: `/v1/books` و `/v2/books`، مع مكان النسخة الذي يحدده Nest بعد البادئة العامة وقبل مسار المتحكم.

## المشكلة التي يحلها

العميل القديم يثبت العنوان في التطبيق ولا يرسل ترويسات إضافية. يحتاج أن يرى النسخة في الشريط. في الوقت نفسه لا تريد شرط `if (url.startsWith('/v1'))` داخل كل Handler.

## الفكرة ببساطة

`VersioningType.URI` يأخذ النسخة من المسار. تكتب `@Controller({ version: '1' })` أو `@Version('2')` على المسار. Nest يدرج `v` قبل الرقم ما لم تغيّر `prefix`. الطلب الذي لا يطابق نسخة معلنة يعيد 404.

## التعريف التقني

```ts
app.enableVersioning({ type: VersioningType.URI });
```

`VersioningType` من `@nestjs/common`. البادئة الافتراضية `v`، فالنسخة `'1'` تصبح `/v1/...`. غيّرها بـ `prefix` أو عطّلها بـ `prefix: false`. النسخة تُوضع بعد `setGlobalPrefix` إن وُجدت، ثم يأتي مسار المتحكم. `@Version` على الدالة يتجاوز نسخة المتحكم. مصفوفة `version: ['1', '2']` تخدم الدالة نفسها للنسختين. `VERSION_NEUTRAL` مسار بلا مقطع نسخة في URI، ويلتقط الطلب أيًا كانت نسخته بما فيها غياب النسخة. `defaultVersion: '1'` يجعل المسار الذي لم يصرح بنسخة يتصرف كـ v1.

## Mental Model

```text
/api   +  /v1  +  /books/:id
بادئة     نسخة    مسار المتحكم
```

## أبسط مثال

```ts
import { Controller, Get } from '@nestjs/common';
import { BooksService } from './books.service.js';
import { toV1 } from './book.presenters.js';

@Controller({ path: 'books', version: '1' })
export class BooksV1Controller {
  constructor(private readonly books: BooksService) {}

  @Get(':id')
  async find(@Param('id') id: string) {
    return toV1(await this.books.find(id));
  }
}
```

أضف `Param` إلى الاستيراد من `@nestjs/common`. المسار الفعلي `/v1/books/:id`.

## شرح المثال

- `'1'` بلا حرف `v`. الحرف يأتي من `prefix`.
- الخدمة تعيد النموذج الداخلي. `toV1` وحدها تعرف `price` كعدد.
- متحكم ثانٍ `version: '2'` يستدعي `toV2` على نفس الخدمة.

## ماذا يحدث داخليًا؟

1. `enableVersioning` يغيّر بناء المسارات عند الإقلاع.
2. طلب `GET /v1/books/b1` يطابق متحكم النسخة 1.
3. طلب `GET /books/b1` لا يطابق إن لم يوجد `defaultVersion` أو `VERSION_NEUTRAL`، والنتيجة 404 كما في الصفحة.
4. طلب `GET /v3/books/b1` بلا متحكم v3 يعيد 404 أيضًا.
5. مع `setGlobalPrefix('api')` يصبح العنوان `/api/v1/books/b1` لا `/v1/api/books/b1`.

## مثال عملي

نسختان ودالة صحة محايدة:

```ts
import { Controller, Get, Version, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({ version: '2', path: 'books' })
export class BooksV2Controller {
  @Get()
  list() {
    return [{ id: 'b1', price: { amount: 2500, currency: 'SAR' } }];
  }

  @Version('1')
  @Get('legacy-count')
  count() {
    return { count: 1 };
  }
}

@Controller({ version: VERSION_NEUTRAL })
export class HealthController {
  @Get('health')
  live() {
    return { ok: true };
  }
}
```

`legacy-count` على متحكم v2 يتصرف كـ v1 لأن `@Version('1')` يتجاوز نسخة الصنف. الصحة على `/health` بلا مقطع نسخة.

## أخطاء شائعة

- كتابة `version: 'v1'` فتبحث Nest عن مقطع `vv1` مع البادئة الافتراضية.
- توقع أن البادئة العالمية تأتي بعد رقم النسخة.
- نسخ `BooksService` لكل إصدار.
- نسيان اختبار 404 للنسخة غير الموجودة، فيُظن أن أي رقم يصل إلى v1.

## متى نستخدمه؟

عندما يجب أن يظهر العقد في العنوان: جوال لا تضبط ترويساته، أو وكيل يوجّه حسب بادئة المسار، أو توثيق عام يُقرأ من غير مجموعة ترويسات.

## متى لا نستخدمه؟

عندما تريد إبقاء المسار ثابتًا وتبديل التمثيل بتفاوض المحتوى. ذلك Media Type. وعندما تكسر كل الإشارات المرجعية المخزنة عند كل إصدار بلا داعٍ.

## مقارنة

| الطلب | مع `version: '1'` على books | مع `defaultVersion: '1'` لمسار بلا نسخة |
| --- | --- | --- |
| `/v1/books` | يطابق | يطابق المصرح به |
| `/books` | 404 | يطابق الافتراضي |
| `/v9/books` | 404 | 404 إن لم تُصرَّح 9 |

## تمارين

### تمرين سهل

المتحكم `version: '2'` والمسار `books`. ما عنوان القائمة؟

<details>
<summary>الحل</summary>

`/v2/books` مع البادئة الافتراضية `v`.

</details>

### تمرين متوسط

`setGlobalPrefix('api')` و `prefix: false` والنسخة `'1'`. كيف يصير المسار؟

<details>
<summary>الحل</summary>

`/api/1/books` لأن البادئة العالمية أولًا، و `prefix: false` يلغي حرف `v`، وتبقى النسخة قبل مسار المتحكم.

</details>

### تمرين متقدم

صحة Terminus يجب أن تبقى `/health` للموازن الذي لا يعرف النسخ، بينما الكتب تتغير. أي رمز نسخة للصحة؟

<details>
<summary>الحل</summary>

`VERSION_NEUTRAL` على متحكم الصحة. الصفحة تقول إن المورد المحايد في URI لا يظهر فيه مقطع النسخة، ويطابق الطلب بنسخة أو بلاها.

</details>

## أسئلة مراجعة

- من أين يأتي حرف `v`؟
- أين تقع النسخة بالنسبة لـ `setGlobalPrefix`؟
- ماذا يتجاوز `@Version` على الدالة؟
- متى تعيد Nest 404 رغم وجود الدالة في الملف؟

## الخلاصة

URI Versioning يضع العقد في المسار بعد البادئة العامة وقبله مسار المتحكم. الرقم في الكود بلا `v` إلا إذا غيّرت `prefix`. المسار بلا نسخة 404 إلى أن تضع افتراضيًا أو `VERSION_NEUTRAL`.

## ماذا بعد؟

بعض عملاء رف يثبتون `/books` في العناوين المخزنة ويريدون النسخة في ترويسة. الدرس التالي Header Versioning.

## المصادر الرسمية

- NestJS — Versioning: https://docs.nestjs.com/techniques/versioning
- NestJS — Global prefix: https://docs.nestjs.com/faq/global-prefix
