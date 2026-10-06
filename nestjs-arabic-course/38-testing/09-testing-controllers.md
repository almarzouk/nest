# اختبار Controllers

## الهدف من هذا الدرس

اختبار أن المتحكم يمرر المعرف والجسم إلى الخدمة ويعيد ناتجها، مع خدمة وهم، بلا إثبات SQL.

## المشكلة التي يحلها

المتحكم الرقيق سهل الكسر بصمت: يقرأ `body.title` بينما الخدمة تنتظر `input`، أو ينادي `publish` بلا `@Param`. اختبار الخدمة لا يرى ذلك. اختبار المتحكم يستبدل الخدمة ويتفقد الوسائط. هو لا يستبدل رحلة Supertest عندما يكون الأنبوب والحارس جزءًا من العقد.

## الفكرة ببساطة

`TestingModule` فيها المتحكم و `overrideProvider(BooksService)` أو `useValue` من البداية. `get(BooksController)` ثم استدعاء الدالة مباشرة. التوقع على الوهم وعلى القيمة المعادة. الزينة `@Post` لا تُنفَّذ بهذا الاستدعاء. لذلك هذا الملف للتمرير لا لرمز HTTP.

## التعريف التقني

```ts
import { Test } from '@nestjs/testing';
import { BooksController } from './books.controller.js';
import { BooksService } from './books.service.js';

const books = {
  publish: vi.fn(async () => ({ id: '42', status: 'PUBLISHED' })),
};

const moduleRef = await Test.createTestingModule({
  controllers: [BooksController],
  providers: [{ provide: BooksService, useValue: books }],
}).compile();

const controller = moduleRef.get(BooksController);
const result = await controller.publish('42');
expect(books.publish).toHaveBeenCalledWith('42');
expect(result.status).toBe('PUBLISHED');
```

`controller.publish('42')` يمرر الوسيط كما ستمرره الزينة إن كان المعامل الأول هو `@Param('id')`. إن كان ترتيب المعاملات في الدالة `(body, id)` والاختبار ينادي `(id)` فأنت تختبر نداءً لا يشبه الإطار. عندها فضّل رحلة HTTP. استدعاء الدالة مناسب عندما يكون التوقيع `(id: string)` واضحًا.

لا تضع Prisma في `providers` هنا. إن طُلبت من المتحكم فالتصميم يعيد منطق القاعدة إلى الطبقة الخطأ.

## Mental Model

```text
وهم BooksService
  BooksController.publish(id)
    توقع publish(id)
```

## أبسط مثال

إنشاء:

```ts
await controller.create({ title: 'NestJS' });
expect(books.create).toHaveBeenCalledWith({ title: 'NestJS' });
```

المتحكم لا يضيف `userId` من الجسم. إن كان المعرف من الطلب، الاختبار الذي ينادي الدالة مباشرة لا يرى الحارس. اختبر استخراج المستخدم في رحلة أو بتمرير كائن طلب إن كان المعامل `@Req()` ظاهرًا، والأوضح إبقاء المتحكم على `@CurrentUser()` واختبار الزينة في e2e.

## شرح المثال

الخدمة الوهم تعيد كائنًا والاختبار يتحقق أن المتحكم لم يبدّل `status`. إن كان المتحكم يغلّف الرد في `{ data }` فهذا مكان التوقع. لا تتوقع `logger` ولا شكل Prisma.

## ماذا يحدث داخليًا؟

الحاوية تنشئ المتحكم وتحقن الوهم. نداء الدالة نداء TypeScript عادي. `ValidationPipe` لا يعمل. `ParseUUIDPipe` لا يعمل. لذلك لا تختبر هنا «سلسلة فارغة تصبح 400». ضع ذلك في الرحلة حيث السلسلة موجودة. ما تثبته هنا: لا حساب ISBN داخل المتحكم، بل نداء واحد للخدمة.

## مثال عملي

متحكم الغلاف بعد الرفع لا يصل إلى هذا الملف بملف حقيقي. مرر كائنًا يشبه الجزء الذي تقرأه الدالة:

```ts
await controller.replace('42', { filename: 'a.png' } as Express.Multer.File);
expect(books.setCoverKey).toHaveBeenCalledWith('42', 'a.png');
```

إن استدعت الدالة `file.buffer` فالاختبار يذكرك أن المتحكم يلمس البايتات. انقل ذلك إلى الخدمة أو إلى المنتج.

## أخطاء شائعة

- توقع `400` من نداء الدالة بلا أنبوب.
- بناء Prisma حقيقي خلف المتحكم.
- اختبار كل فروع `publish` هنا بدل الخدمة.
- نسيان أن ترتيب المعاملات ليس ترتيب الزينة إلا بالانضباط.

## متى نستخدمه؟

للمتحكم الرقيق: المعامل الصحيح يصل إلى الخدمة والناتج يُعاد.

## متى لا نستخدمه؟

عندما يكون السؤال عن الرمز أو الأنبوب أو الحارس. انتقل إلى Supertest. وإن صار المتحكم سمينًا، لا تختبر السمنة. انقلها.

## مقارنة

| السؤال | الأداة |
| --- | --- |
| هل نُودي `publish` بالمعرف؟ | هذا الدرس |
| هل الجسم الباطل `400`؟ | رحلة HTTP |
| هل الصف تغيّر؟ | تكامل الخدمة |

## تمارين

### تمرين سهل

ماذا تضع مكان `BooksService` في الحاوية؟

<details>
<summary>الحل</summary>

`useValue` بكائن فيه الدوال التي يناديها المتحكم.

</details>

### تمرين متوسط

لماذا لا يُختبر `ValidationPipe` بنداء `controller.create`؟

<details>
<summary>الحل</summary>

لأن النداء المباشر لا يمر على أنابيب الإطار. الأنبوب يعمل على مسار HTTP.

</details>

### تمرين متقدم

اختبار الغلاف يمرر `buffer`. ماذا يعني ذلك للتصميم؟

<details>
<summary>الحل</summary>

أن المتحكم صار يتعامل مع البايتات. مرر المفتاح فقط إلى الخدمة حتى يبقى الاختبار على `filename` أو مفتاح مولَّد.

</details>

## أسئلة مراجعة

- ماذا لا يشغّله `get(Controller)` ثم النداء؟
- أي صنف لا يُسجَّل حقيقيًا هنا؟
- متى تترك هذا الملف إلى Supertest؟

## الخلاصة

اختبار المتحكم وهم خدمة ونداء مباشر يثبت التمرير والإرجاع. الرمز والأنبوب والحارس رحلة لاحقة.

## ماذا بعد؟

Supertest على `getHttpServer()` لقراءة `201` و `400` فعلًا.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
