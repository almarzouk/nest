# Layered Architecture

## الهدف من هذا الدرس

ترتيب كود Nest في طبقات اتجاهها واحد: مدخل HTTP، ثم حالة الاستخدام، ثم التخزين، حتى لا يصل Prisma إلى المتحكم ولا يصل `Response` إلى قاعدة البيانات.

## المشكلة التي يحلها

متحكم يستدعي `prisma.book.create` ويتحقق من الدور ويحسب الخصم يصعب اختباره بلا شبكة وقاعدة، وأي تغيير في الجدول يمس HTTP. الطبقات تسمي مكان كل تغيير. Nest لا يفرض الأسماء، ويمنح أماكن طبيعية: Controller و Service و Provider للتخزين.

## الفكرة ببساطة

الطلب يدخل من الطبقة الخارجية. كل طبقة تنادي التي تحتها بعقد صغير وترجع نتيجة. الطبقة السفلى لا تستورد المتحكم ولا `@Req()`. في مشروع الكتب الآن: طبقة مدخل، طبقة تطبيق، طبقة بيانات. ثلاث تكفي.

## التعريف التقني

طبقة المدخل: `BooksController` يقرأ DTO ويعيد شكل HTTP. لا يعرف أعمدة SQL.

طبقة التطبيق: `BooksService` تقرر إنشاء الكتاب وتتحقق من قاعدة العمل (العنوان غير فارغ بعد أن مر Pipe، الخصم لا يتجاوز السعر). تستقبل أنواعًا plain و `userId` نصًا.

طبقة البيانات: `PrismaService` أو دالة تغلف الاستعلام. تعرف الجدول وتخفي `select` عن الخدمة قدر الحاجة.

الاتجاه: Controller → Service → Prisma. السهم العكسي ممنوع. الاستثناءات التي ترميها الخدمة يمكن أن تكون أخطاء دومين يترجمها Filter إلى HTTP، لا أن تستورد الخدمة `Response`.

هذا ليس كل Clean Architecture. لا كيانات غنية ولا منافذ بعد. إنه ترتيب اعتماد يناسب معظم وحدات Nest في هذه الدورة.

## Mental Model

```text
HTTP DTO
  -> BooksController
  -> BooksService
  -> PrismaService
  -> جدول books
الرجوع: سجل -> كائن -> JSON
```

## أبسط مثال

```ts
import { Body, Controller, Post } from '@nestjs/common';
import { BooksService } from './books.service.js';
import { CreateBookDto } from './create-book.dto.js';

@Controller('books')
export class BooksController {
  constructor(private readonly books: BooksService) {}

  @Post()
  create(@Body() dto: CreateBookDto) {
    return this.books.create(dto);
  }
}
```

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service.js';
import { CreateBookDto } from './create-book.dto.js';

@Injectable()
export class BooksService {
  constructor(private readonly prisma: PrismaService) {}

  create(dto: CreateBookDto) {
    return this.prisma.book.create({ data: { title: dto.title, priceMinor: dto.priceMinor } });
  }
}
```

## شرح المثال

المتحكم لا يرى `data:`. الخدمة لا ترى `@Body`. تغيير اسم العمود يمس الخدمة أو طبقة البيانات، لا توقيع المسار إن بقي DTO ثابتًا. إن كبر `create` فانقل SQL إلى دالة `BooksStore` وأبقِ الخصم في الخدمة. تلك ما زالت طبقات ثلاثًا بأسماء أوضح.

## ماذا يحدث داخليًا؟

1. المسار يضرب المتحكم فقط.
2. الحقن يعطي الخدمة نسخة Singleton.
3. الخدمة تنادي العميل.
4. الكائن الراجع يصعد. طبقة HTTP في Nest تحوله إلى JSON.
5. فشل القيد في القاعدة يستثنى في الأسفل ويُترجم في Filter أعلى، لا داخل `prisma` بتفاصيل المسار.
6. اختبار الخدمة يمرر عميلًا مزيفًا بلا `listen`.

## مثال عملي

وحدة طلبات: `OrdersController` يستخرج `userId` من الديكور. `OrdersService.place` يتحقق أن السلة غير فارغة ثم ينادي `prisma.order.create`. لا تضع التحقق في المتحكم بعد وجود Pipe للشكل، ولا في استدعاء Prisma الخام إن كانت القاعدة ستتكرر من مهمة طابور لاحقة. الطابور يستدعي الخدمة نفسها لا المتحكم.

## أخطاء شائعة

- `PrismaClient` داخل المتحكم.
- تمرير `Request` إلى الخدمة.
- طبقة «Manager» رابعة بلا فرق عن الخدمة.
- جعل الطبقة السفلى ترمي `HttpException` لكل مخالفة قيد، فتُربط القاعدة بـ HTTP.
- مجلدات layers على مستوى المستودع كله بدل وحدة لكل ميزة. الحدود بين الميزات درس المعمارية النمطية.

## متى نستخدمه؟

أي ميزة CRUD في هذه الدورة: مدخل، تطبيق، بيانات.

## متى لا نستخدمه؟

لا تزد طبقة لأن رسمًا في مقال فيه خمس طبقات. أضف طبقة عندما يوجد تغيير مستقل، مثل استعلام معقد يتكرر من أكثر من خدمة.

## مقارنة

| الطبقة | تعرف | لا تعرف |
| --- | --- | --- |
| Controller | مسار، DTO، رمز HTTP | جداول |
| Service | قواعد العمل | `Response` |
| Prisma | الاستعلام | شكل جسم HTTP |

## تمارين

### تمرين سهل

من يحق له استدعاء `prisma.book.create` في الترتيب أعلاه؟

<details>
<summary>الحل</summary>

`BooksService` أو طبقة بيانات تناديها الخدمة، لا المتحكم.

</details>

### تمرين متوسط

مهمة طابور تريد إنشاء كتاب. أي طبقة تستدعي؟

<details>
<summary>الحل</summary>

`BooksService.create`، حتى لا تعتمد المهمة على HTTP ولا تنسخ SQL.

</details>

### تمرين متقدم

قيد فريد على `title` فشل. أين يُترجم إلى `409`؟

<details>
<summary>الحل</summary>

في Exception Filter أو ترجمة عند حافة HTTP، لا داخل `PrismaService` التي قد تُستدعى من غير HTTP.

</details>

## أسئلة مراجعة

- ما اتجاه الاستيراد؟
- ماذا يستقبل الدومين من الهوية: `Request` أم `userId`؟
- متى تضيف طبقة رابعة؟

## الخلاصة

ثلاث طبقات باتجاه واحد تكفي لمعظم وحدات Nest الآن: مدخل، خدمة، تخزين. امنع السهم العكسي قبل أن تسمي المعمارية.

## ماذا بعد؟

Modular Architecture: الحد بين الميزات، لا بين الطبقات فقط.

## المصادر الرسمية

- NestJS — Modules: https://docs.nestjs.com/modules
- NestJS — Controllers: https://docs.nestjs.com/controllers
