# Clean Architecture

## الهدف من هذا الدرس

قراءة قاعدة الاعتماد نحو الداخل: سياسة العمل لا تستورد تفاصيل HTTP أو Prisma، والحكم متى تطبقها في Nest ومتى تبقى الطبقات الثلاث كافية.

## المشكلة التي يحلها

خدمة «تطبيق» تستورد `@nestjs/common` و `PrismaClient` معًا ليست مستقلة عن الإطار ولا عن القاعدة. نقلها إلى عامل أو اختبار بلا Nest مكلف. Clean Architecture، كما صاغها Robert C. Martin، تجعل الكيانات وحالات الاستخدام في الداخل، وتجعل الواجهات والبنية في الخارج. الداخل لا يعرف الخارج. الخارج يحقن تنفيذًا يطبّق منفذًا يعرفه الداخل.

## الفكرة ببساطة

دوائر: كيانات في القلب، ثم حالات استخدام، ثم محولات. السهم يشير إلى الداخل فقط. في Nest المحول الخارجي متحكم. حالة الاستخدام خدمة بلا ديكور HTTP. المنفذ واجهة أو رمز حقن. Prisma محول في الدائرة الخارجية يطبّق المنفذ. الإطار يوصل المحول بالمنفذ عبر `provide` و `useClass`.

## التعريف التقني

لا حزمة `@nestjs/clean`. القرار هيكلي.

كيان `Book`: حقول وقواعد لا تعرف SQL، مثل رفض سعر سالب. قد يكون صنفًا بلا `@Injectable` إن لم يحتج الحاوية.

حالة استخدام `CreateBook`: تنادي `BooksPort.save`. لا تستورد `@prisma/client`.

منفذ `BooksPort` في الداخل. محول `PrismaBooksAdapter` في مجلد بنية يستورد المنفذ وينفذه.

الرمز من فصل المزودات المخصصة:

```ts
{ provide: BOOKS, useClass: PrismaBooksAdapter }
```

`CreateBook` يحقن `BOOKS`. المتحكم يحقن `CreateBook` فقط.

قاعدة الاعتماد: ملف حالة الاستخدام لا يظهر في `import` له مسار `prisma` أو `express`. ملف المحول يجوز أن يستورد الاثنين.

في تطبيق CRUD صغير، كيان بلا سلوك وواجهة تعيد تمرير Prisma تزيد ملفاتًا بلا حماية. طبّق الدوائر عندما السياسة ستعيش أطول من القاعدة أو من HTTP، أو عندما الاختبار يجب أن يشغّل الحالة بلا Docker. غير ذلك الطبقات داخل الوحدة النمطية تكفي، وهذا ما يقوله درس «ما لا تستورده».

## Mental Model

```text
CreateBook  ->  BooksPort  <-  PrismaBooksAdapter
     ^
BooksController
```

السهمان نحو المنفذ. المتحكم لا يقفز إلى المحول.

## أبسط مثال

```ts
export interface BooksPort {
  save(input: { title: string; priceMinor: number }): Promise<{ id: string }>;
}

export const BOOKS = Symbol('BOOKS');
```

```ts
import { Inject, Injectable } from '@nestjs/common';
import { BOOKS, type BooksPort } from './books.port.js';

@Injectable()
export class CreateBook {
  constructor(@Inject(BOOKS) private readonly books: BooksPort) {}

  execute(input: { title: string; priceMinor: number }) {
    if (input.priceMinor < 0) {
      throw new Error('السعر السالب مرفوض');
    }
    return this.books.save(input);
  }
}
```

## شرح المثال

`execute` سياسة. الفحص بلا Prisma. `Error` هنا تعليمي؛ في المشروع استثناء دومين يترجمه Filter. المحول لاحقًا يحوّل الصف إلى `{ id }`. تغيير Prisma إلى استعلام آخر يمس المحول والربط في الوحدة، لا `CreateBook`.

## ماذا يحدث داخليًا؟

1. المتحكم يستقبل DTO وينادي `execute`.
2. الحاوية تمرر المحول خلف `BOOKS`.
3. السياسة تفحص ثم تطلب `save`.
4. المحول يكتب SQL أو استدعاء Prisma.
5. الناتج يصعد كائنًا plain.
6. اختبار `CreateBook` يمرر `{ save: async () => ({ id: '1' }) }` بـ `useValue` على `BOOKS`.

## مثال عملي

نقل `CreateBook` إلى عامل طابور: العامل يستدعي `execute` بعد أن تبني الوحدة المحول نفسه أو محولًا يتصل بقاعدة العامل. لا تنسخ فحص السعر في معالج الطابور. إن كان الفحص في المتحكم ضاع عند النقل. هذا ربح الدائرة الداخلية.

## أخطاء شائعة

- واجهة `save` تعيد نوع `Book` من Prisma فيتسرب العميل إلى السياسة.
- كيان فيه `@Inject` لـ Prisma.
- مجلد `domain` يستورد `PrismaService`.
- تطبيق الدوائر على `GET /health`.
- تسمية كل خدمة `UseCase` من غير منفذ، فتبقى الخدمة تستورد Prisma والاسم فقط نظيف.

## متى نستخدمه؟

سياسة ستُختبر بلا قاعدة، أو أكثر من محول دخول (HTTP وطابور) لنفس القاعدة.

## متى لا نستخدمه؟

CRUD يقرأ جدولًا ويعرضه. طبقة خدمة داخل `BooksModule` أوضح حتى يظهر ألم حقيقي: اختبار بطيء أو محول ثانٍ.

## مقارنة

| الأسلوب | الداخل يعرف Prisma؟ | كلفة الملفات |
| --- | --- | --- |
| طبقات بسيطة | الخدمة تعرف العميل | منخفضة |
| Clean | حالة الاستخدام تعرف المنفذ | منفذ ومحول لكل حالة تستحق |

## تمارين

### تمرين سهل

إلى أين يشير سهم الاعتماد؟

<details>
<summary>الحل</summary>

نحو الداخل: المحول يطبّق منفذًا يعرفه الداخل، والسياسة لا تستورد المحول.

</details>

### تمرين متوسط

لماذا إعادة نوع Prisma من `BooksPort` تكسر القاعدة؟

<details>
<summary>الحل</summary>

لأن السياسة التي تستعمل الناتج تستورد تفاصيل القاعدة. المنفذ يعيد أشكالًا يملكها الداخل.

</details>

### تمرين متقدم

`GET` الصحة يفحص العملية فقط. لماذا لا تضع له منفذ `HealthPort` في القلب؟

<details>
<summary>الحل</summary>

لا سياسة عمل فيه. متحكم وIndicator من Terminus في الحافة يكفيان. الدوائر تحمي سياسة تتغير، لا كل ملف.

</details>

## أسئلة مراجعة

- ما دور `Symbol` هنا؟
- ماذا يُختبَر بـ `useValue`؟
- متى الطبقات الثلاث أصدق؟

## الخلاصة

Clean Architecture تعني أن السياسة لا تستورد الإطار ولا القاعدة، والمحولات في الحافة تطبّق منافذ. طبّقها حيث يوجد ألم نقل أو اختبار، لا كقالب لكل مسار.

## ماذا بعد؟

Hexagonal Architecture: الأسماء Ports و Adapters لنفس اتجاه الاعتماد.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
- Robert C. Martin، Clean Architecture (الكتاب مصدر الفكرة، والتطبيق هنا على وحدات Nest)
