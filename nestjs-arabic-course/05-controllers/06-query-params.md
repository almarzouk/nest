# Query Params

## الهدف من هذا الدرس

قراءة معاملات الاستعلام بـ `@Query`: مفتاح واحد أو الكائن كاملًا.

## المشكلة التي يحلها

تصفية الفهرس بالصفحة والموضوع تغيّر نتيجة `GET /catalog` من غير أن تغيّر هوية المورد. وضع `page` داخل المسار يصنع عنوانًا جديدًا لكل صفحة. الاستعلام وُجد لهذا التضييق.

## الفكرة ببساطة

`GET /catalog?topic=history&page=2` يطابق `@Get()` على بادئة `catalog`. `@Query('topic')` يعيد `"history"`. `@Query()` يعيد كل المفاتيح.

## التعريف التقني

`@Query(key)` يقرأ `req.query[key]`. `@Query()` يقرأ `req.query`. القيم نصوص. النوع `number` على الوسيط لا يحوّل `"2"`. التحويل Pipe مثل `ParseIntPipe`، وفصله لاحق. إلى ذلك الحين حوّل في الخدمة تحويلًا ظاهرًا أو أبقِ النص.

الاستعلام المعشّق والمصفوفات:

```text
?filter[where][topic]=history
?tag[]=atlas&tag[]=map
```

تحتاج محلّل استعلام يفهم التداخل. على Express:

```ts
const app = await NestFactory.create<NestExpressApplication>(AppModule);
app.set('query parser', 'extended');
```

على Fastify يُمرَّر `querystringParser` إلى `FastifyAdapter`، وحزمة `qs` تحلّل التداخل بعد `npm install qs`. من غير هذا تبقى المفاتيح نصوصًا مسطحة.

`@Query` غير `@QueryMethod`. الثاني طريقة HTTP نادرة، والأول معامل `?`.

## Mental Model

```text
GET /catalog/maps?page=2&topic=history
  المسار يختار @Get('maps')
  @Query('page')  -> "2"
  @Query('topic') -> "history"
```

غياب `topic` يجعل `@Query('topic')` قيمة `undefined`.

## أبسط مثال

```ts
import { Controller, Get, Query } from '@nestjs/common';
import { CatalogService } from './catalog.service.js';

@Controller('catalog')
export class CatalogController {
  constructor(private readonly catalog: CatalogService) {}

  @Get()
  findAll(@Query('topic') topic?: string, @Query('page') page?: string) {
    return this.catalog.search(topic, page);
  }
}
```

## شرح المثال

- المسار ثابت: `GET /catalog`. التصفية كلها في الاستعلام.
- `page` يصل نصًا حتى لو كتب العميل `page=2`.
- العلامتان `?` على نوع TypeScript تصفان الغياب للمترجم. وقت التشغيل القيمة `undefined` عند غياب المفتاح، وهذا مستقل عن الـ Decorator.
- الخدمة تقرر الصفحة الافتراضية عندما يكون `page` غائبًا.

## ماذا يحدث داخليًا؟

المحوّل يملأ `req.query` قبل المعالج. Nest تمرر الحقل المطلوب أو الكائن. ترتيب الوسائط في الدالة حر؛ الاسم داخل `@Query('page')` هو المفتاح.

كائن كامل:

```ts
@Get()
findAll(@Query() query: { topic?: string; page?: string }) {
  return this.catalog.search(query.topic, query.page);
}
```

المفاتيح الزائدة تبقى على الكائن. Pipe لاحق يمكن أن يقصر الشكل. هذا الدرس لا يرفض مفتاحًا مجهولًا.

## مثال عملي

```ts
import { Controller, Get, Param, Query } from '@nestjs/common';
import { CatalogService } from './catalog.service.js';

@Controller('catalog')
export class StacksController {
  constructor(private readonly catalog: CatalogService) {}

  @Get(':stack/items')
  items(
    @Param('stack') stack: string,
    @Query('page') page = '1',
  ) {
    return this.catalog.page(stack, page);
  }
}
```

`GET /catalog/maps/items?page=2`: المسار يثبت الرف `maps`، والاستعلام يثبت الصفحة. القيمة الافتراضية `'1'` في الوسيط تعمل لأن `@Query` يعيد `undefined` عند الغياب، فيأخذ المعامل الافتراضي في JavaScript دوره. إن وصل `page` فارغًا كنص `""` فالافتراضي لا يعمل لأن النص الفارغ قيمة موجودة.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| مقارنة `page === 2` | القيمة `"2"` والمقارنة لا تنجح |
| قراءة `req.query` يدويًا بجانب `@Query` من غير سبب | مصدران للحقيقة نفسها في الدالة |
| توقع `filter[where][topic]` ككائن من غير محلّل `extended` | المفتاح يصل مسطحًا أو يضيع حسب المحول |
| وضع معرف المورد الوحيد في الاستعلام | `GET /catalog?id=9` يخفي هوية العنصر خارج المسار |

## متى نستخدمه؟

للتصفية والفرز والترقيم على مسار ثابت أو على مجموعة محددة بمعامل مسار: `topic` و `page` و `sort`.

## متى لا نستخدمه؟

لهوية عنصر واحد تعاد زيارتها. تلك `:id` في المسار. ولجسم إنشاء JSON. ذلك `@Body` على `@Post`.

## مقارنة

| المدخل | مكانه في الطلب |
| --- | --- |
| `@Param('stack')` | قطعة من المسار |
| `@Query('page')` | مفتاح بعد `?` |
| `@Body()` | جسم الطلب، غالبًا JSON |

## تمارين

### تمرين سهل

ماذا تعيد `@Query('topic')` لـ `GET /catalog?topic=history`؟

<details><summary>الحل</summary>

النص `"history"`.

</details>

### تمرين متوسط

لماذا `page: number` على الوسيط لا يكفي لجمع `page + 1`؟

<details><summary>الحل</summary>

القيمة وقت التشغيل نص. `"2" + 1` يلصق إلى `"21"`. التحويل يتم بـ Pipe لاحق أو بدالة ظاهرة مثل `Number(page)` بعد التحقق من الغياب.

</details>

### تمرين متقدم

`GET /catalog/maps/items` من غير `page` يدخل الفرع الافتراضي، بينما `?page=` لا يفعل. ما السبب في مثال الرفوف؟

<details><summary>الحل</summary>

الغياب يعطي `undefined` فيأخذ المعامل الافتراضي `'1'`. `?page=` يمرر نصًا فارغًا، والنص الفارغ لا يستدعي الافتراضي في توقيع JavaScript.

</details>

## أسئلة مراجعة

- ما الفرق بين `@Query('page')` و `@Query()`؟
- ما نوع قيمة الاستعلام قبل أي Pipe؟
- متى تحتاج `query parser` بقيمة `extended`؟

## الخلاصة

`@Query` يقرأ ما بعد `?` كنصوص. المسار يختار المعالج، والاستعلام يضيّق النتيجة، والتحويل والتحقق يأتيان مع Pipes.

## ماذا بعد؟

جسم الطلب: Body.

## المصادر الرسمية

- NestJS — Query parameters: https://docs.nestjs.com/controllers#query-parameters
