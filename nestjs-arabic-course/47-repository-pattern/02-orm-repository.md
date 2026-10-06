# ORM Repository

## الهدف من هذا الدرس

تمييز `Repository` الذي يقدّمه ORM، خصوصًا TypeORM، عن مستودع الدومين: الأول مساعد وصول للجدول، والثاني عقد مجموعة للجذر.

## المشكلة التي يحلها

توثيق TypeORM يحقن `Repository<User>` وفيه `find` و `save` و `createQueryBuilder`. الاسم يطابق النمط فيُظن أن حقن هذا الصنف يحقق العزل. الصنف يعرف كيان ORM والأعمدة، ويعيد كائنات مرتبطة بالجلسة أو بالجدول. خدمة الدومين التي تستقبله تظل مرتبطة بالأداة. ذلك مفيد كطبقة بيانات داخل Nest. هو ليس المنفذ الذي يمنع Prisma أو TypeORM من الظهور في السياسة.

## الفكرة ببساطة

`@InjectRepository(BookEntity)` يعطيك أدوات الجدول `book`. استعمله في محول أو في خدمة وحدة صغيرة قررت عدم العزل. لا تمرره إلى `PlaceOrder` التي يُفترض أنها أعمى، ولا تعِد كيان TypeORM من واجهة تسميها منفذ دومين. Prisma ليس فيه الصنف نفسه، وعميل `PrismaClient` يلعب دور المساعد نفسه إن استُعمل مباشرًا: وصول بيانات قوي، لا مجموعة دومين.

## التعريف التقني

شكل Nest مع TypeORM، كما في توثيق قاعدة البيانات للأداة:

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { BookEntity } from './book.entity.js';

@Injectable()
export class BooksStore {
  constructor(
    @InjectRepository(BookEntity)
    private readonly books: Repository<BookEntity>,
  ) {}

  findEntity(id: string): Promise<BookEntity | null> {
    return this.books.findOne({ where: { id } });
  }
}
```

`Repository<BookEntity>` هنا تفصيل. `BooksStore` اسم صادق. إن سمّيته `BooksRepository` فوثّق أنه مستودع ORM لا منفذ الدومين، أو حوّله في الدرس التالي إلى محول يركّب كيانًا خاصًا بك.

العمليات `find({ where, relations, take })` استعلامات. `save(entity)` يحفظ صف الكيان كما يعرّفه ORM، وقد يكتب أعمدة لم تمر بدوال جذر. هذا يختلف عن `save(order)` الذي يفترض أن `order` حُمي مسبقًا.

Prisma:

```ts
this.prisma.book.findUnique({ where: { id } });
```

نفس الدور: مساعد. وضعه خلف منفذ قرار إضافي. حقنه في خدمة الكتالوج البسيطة متوافق مع «ما لا تستورده» حتى يظهر الألم.

لا تبنِ `class GenericRepository<T>` يلف كل الجداول. التوثيق والألم اليومي في الاستعلامات المختلفة، والتجريد العام يخفيها خلف `any`.

## Mental Model

```text
Repository<BookEntity>  -> جدول book عبر TypeORM
PrismaClient.book       -> جدول book عبر Prisma
OrdersRepository الدومين -> مجموعة Order بلا أداة
```

الأولان يُحقنان حيث قبلت التفصيل. الثالث يُحقن في السياسة.

## أبسط مثال

`BooksStore.findEntity` أعلاه داخل `CatalogModule`. المتحكم ينادي خدمة تنادي `BooksStore`. لا واجهة `BooksPort` بعد. الاسم لا يقول دومين. عند اختبار HTTP يمكنك قاعدة اختبار أو مزيف لـ `BooksStore` إن غطيت الدوال التي يناديها المسار.

## شرح المثال

`InjectRepository` تسجيل من `TypeOrmModule.forFeature([BookEntity])`. هذا Dynamic Module من فصل سابق: `forFeature` شريحة على اتصال `forRoot`. الصنف الذي تستلمه يعرف TypeORM. إبقاؤه في طبقة البيانات صدق معماري. تمريره إلى ملف `domain/` يسحب TypeORM إلى الداخل.

## ماذا يحدث داخليًا؟

1. `forRoot` يفتح الاتصال مرة.
2. `forFeature` يسجل `Repository<BookEntity>` في الوحدة.
3. `BooksStore` يستدعي `findOne`.
4. الناتج كيان ORM فيه أعمدة وربما علاقات محمّلة.
5. إرجاعه من Controller يسرّب الشكل إن لم تعِد DTO.
6. استبدال هذا المستودع في اختبار Nest ممكن لأنه مزود، لكن المزيف ما زال يحاكي كيان ORM لا الجذر.

## مثال عملي

تقرير مبيعات يستعمل `createQueryBuilder` على `Repository<OrderEntity>`. هذا مكانه `SalesQuery` في وحدة التقارير، لا دالة على منفذ `Order` التجميعي. القراءة هنا أعمدة ومجاميع. كتابتها عبر `Repository.save` من التقرير تتجاوز جذر الطلب. اترك الكتابة لمالك التجميع حتى لو استعمل هو أيضًا TypeORM داخل المحول.

## أخطاء شائعة

- اعتبار `Repository<Entity>` منفذ Clean تلقائيًا.
- إرجاع `BookEntity` في JSON.
- مستودع عام مولَّد لكل الكيانات.
- `forFeature` في الجذر لكل كيانات التطبيق بدل وحدة الميزة.
- تسمية صنف Prisma `DomainRepository` وهو يعيد نوع العميل.

## متى نستخدمه؟

وصول جدول داخل محول أو خدمة وحدة لم تعزل الدومين، خصوصًا مع TypeORM كما يوثّقه Nest.

## متى لا نستخدمه؟

كعقد لسياسة يجب ألا تستورد الأداة. هناك واجهة تملكها السياسة، والتنفيذ يستعمل مستودع ORM أو Prisma من الداخل.

## مقارنة

| | مستودع ORM | مستودع الدومين |
| --- | --- | --- |
| النوع المعاد | كيان الأداة غالبًا | كيانك أو DTO الدومين |
| من يستورده | طبقة البيانات | السياسة عبر رمز |
| الاستعلام الحر | متاح | خارج المجموعة |
| Prisma | العميل نفسه في هذا الدور | يقف خلف المنفذ |

## تمارين

### تمرين سهل

ماذا يحقن `@InjectRepository(BookEntity)`؟

<details>
<summary>الحل</summary>

`Repository<BookEntity>` من TypeORM، مساعد الجدول لا عقد الدومين تلقائيًا.

</details>

### تمرين متوسط

لماذا تقرير `createQueryBuilder` لا يُضاف إلى `save(order)`؟

<details>
<summary>الحل</summary>

لأن التقرير قراءة أعمدة، بينما `save` يحفظ جذرًا بعد قواعده. الخلط يجعل المجموعة مساعد استعلام عام.

</details>

### تمرين متقدم

خدمة كتالوج صغيرة تستدعي `prisma.book.findMany`. متى تتركها ومتى تلفها؟

<details>
<summary>الحل</summary>

اتركها ما دام لا جذر ولا اختبار يومي بلا قاعدة. لفها بمنفذ عندما تصبح السياسة أوسع من الاستعلام أو عندما تريد مزيفًا على عمليات المجموعة لا على العميل كله.

</details>

## أسئلة مراجعة

- ما دور `forFeature` هنا؟
- لماذا الاسم المشترك يضلل؟
- أين يقف `PrismaClient` من هذا التصنيف؟

## الخلاصة

مستودع ORM أداة جدول يوثّقها TypeORM ويلعب Prisma دورها عند الاستدعاء المباشر. استعمله في الحافة أو في وحدة بسيطة، ولا تعتبره عزل الدومين إلا إذا أخفيت كيانه خلف عقد تملكه السياسة.

## ماذا بعد؟

Domain Repository: الواجهة في الدومين وعميل Prisma خلفها.

## المصادر الرسمية

- NestJS — Database: https://docs.nestjs.com/techniques/database
- TypeORM — Repository API (عبر دليل Nest لقاعدة البيانات)
