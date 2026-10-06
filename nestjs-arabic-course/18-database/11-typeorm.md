# TypeORM كما في التوثيق الرسمي

## الهدف من هذا الدرس

قراءة مسار `@nestjs/typeorm` كما يوثّقه Nest: `forRoot` و `forFeature` ومستودع وكيان، مع إبقاء `synchronize` خارج الإنتاج.

## المشكلة التي يحلها

من يرى Prisma فقط يظن أن أي كيان بزينة هو Prisma. مشاريع قائمة تستخدم TypeORM، وصفحة Nest ما زالت المرجع لذلك التكامل. نسخ مثال البداية كما هو إلى الإنتاج يفتح `synchronize: true`، والتوثيق يحذّر أنه قد يفقد بيانات الإنتاج.

## الفكرة ببساطة

الكيان صنف يصف الجدول. `TypeOrmModule.forRoot` يفتح مصدر البيانات. `forFeature([Book])` يسجّل مستودعًا في الوحدة. الخدمة تحقن `@InjectRepository(Book)`. الترحيلات أداة CLI منفصلة عن حقن Nest، لأن أصناف الترحيل لا تعيش داخل دورة التطبيق.

## التعريف التقني

الحزمة `@nestjs/typeorm` مع `typeorm` ومشغّل القاعدة. مثال الصفحة يستخدم MySQL و `mysql2`. الدورة على PostgreSQL تبدّل `type: "postgres"` ومشغّل `pg`. بقية الشكل واحد.

```ts
TypeOrmModule.forRoot({
  type: "postgres",
  url: process.env.DATABASE_URL,
  autoLoadEntities: true,
  synchronize: false,
});
```

صفحة Nest تعرض `synchronize: true` في مثال البداية، ثم تحذّر: لا تستخدمه في الإنتاج. في هذه الدورة القيمة `false`، والتغيير عبر migrations التي يوفّرها CLI الخاص بـ TypeORM. أصناف الترحيل بلا DI.

`autoLoadEntities: true` يحمّل الكيانات المسجّلة في `forFeature` حتى لا تُسرد كلها في الجذر فتتسرب تفاصيل الوحدات.

```ts
@Entity()
export class Book {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;
}
```

```ts
constructor(
  @InjectRepository(Book)
  private readonly books: Repository<Book>,
) {}
```

`forRootAsync` مع `ConfigService` هو الشكل الذي عاينّاه في فصل الإعداد. `getRepositoryToken(Book)` يبني رمز المزيف في الاختبار.

المعاملات في التوثيق عبر `QueryRunner` أو مدير المعاملة، لا عبر `prisma.$transaction`. لا تخلط الاثنين في المشروع نفسه.

## Mental Model

```text
forRoot            مصدر بيانات التطبيق
forFeature([Book]) Repository في BooksModule
BooksService       this.books.find()
synchronize: false الترحيل خارج الإقلاع
```

## أبسط مثال

```ts
import { Injectable } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { Book } from "./book.entity.js";

@Injectable()
export class BooksService {
  constructor(
    @InjectRepository(Book)
    private readonly books: Repository<Book>,
  ) {}

  findAll() {
    return this.books.find();
  }

  findOne(id: number) {
    return this.books.findOneBy({ id });
  }
}
```

## شرح المثال

`find` و `findOneBy` واجهة المستودع. الكيان يطابق جدولًا. الوحدة التي تملك الخدمة تستورد `TypeOrmModule.forFeature([Book])`. لتستخدم مستودعًا خارجها، أعد تصدير `TypeOrmModule` كما في الصفحة.

هذا المسار مفهوم إن ورثت مشروعًا عليه. مشروع الدورة الجديد يكمل على `PrismaService` الذي سبق بناؤه.

## ماذا يحدث داخليًا؟

`forRoot` ينشئ `DataSource` ويهيّئه عند إقلاع الوحدة ما لم تضبط `manualInitialization`. `synchronize: true` يقارن الكيانات بالقاعدة ويغيّر الجداول عند الإقلاع. هذا مناسب لتجربة سريعة على قاعدة فارغة، وخطر على بيانات قائمة. مع `false` لا يُلمس المخطط حتى تشغّل ترحيل CLI.

`retryAttempts` و `retryDelay` خيارات إضافية على `forRoot` فوق خيارات `DataSource`.

## مثال عملي

خدمة قديمة للإعارات على TypeORM تبقى كما هي إن كان النشر قائمًا عليها. خدمة جديدة في الدورة لا تضاف بجانبها على الجدول نفسه بـ Prisma في الأسبوع نفسه. اختر واحدًا. إن انتقلت، انقل القراءة والكتابة معًا، وأبقِ الترحيلات مصدر شكل الجدول.

اختبار يستبدل المستودع:

```ts
{ provide: getRepositoryToken(Book), useValue: { find: vi.fn() } }
```

## أخطاء شائعة

- `synchronize: true` على الإنتاج.
- سرد كل الكيانات في `AppModule` مع كسر حدود الوحدات، بدل `autoLoadEntities`.
- ترحيل TypeORM يحاول حقن خدمة Nest.
- تشغيل TypeORM و Prisma وكتابة الاثنين على جدول واحد بلا مالك واضح.

## متى نستخدمه؟

مشروع قائم على الكيانات والمستودعات، أو فريق يفضّل مسار Nest الرسمي هذا على التوليد.

## متى لا نستخدمه؟

بجانب Prisma في مكتبة الدورة «للتجربة». مسار واحد يكفي حتى تتقن الترحيل والمعاملات.

## مقارنة

| Prisma في الدورة | TypeORM في التوثيق |
| --- | --- |
| `schema.prisma` وعميل مولَّد | كيان و `Repository` |
| `$transaction` | `QueryRunner` |
| ترحيل Prisma | ترحيل CLI لـ TypeORM |
| بلا `synchronize` | `synchronize: false` عندنا |

## تمارين

### تمرين سهل

ماذا يحقن `@InjectRepository(Book)`؟

<details>
<summary>الحل</summary>

`Repository<Book>` المسجّل عبر `TypeOrmModule.forFeature([Book])`.

</details>

### تمرين متوسط

لماذا مثال التوثيق الذي يضع `synchronize: true` لا يُنسخ إلى الإنتاج؟

<details>
<summary>الحل</summary>

الصفحة تحذّر أن المزامنة قد تفقد بيانات الإنتاج. التغيير يمر بترحيل يُراجع.

</details>

### تمرين متقدم

كيان في `BooksModule` عبر `forFeature`، و `forRoot` بلا `entities` وبلا `autoLoadEntities`. لماذا لا يرى المصدر الجدول؟

<details>
<summary>الحل</summary>

`forFeature` يسجّل المستودع، لكن تحميل الكيان في مصدر البيانات يحتاج إضافتهم إلى `entities` أو `autoLoadEntities: true`.

</details>

## أسئلة مراجعة

- ما دور `forFeature` مقابل `forRoot`؟
- أين تُكتب ترحيلات TypeORM نسبة إلى DI؟
- لماذا لا نخلطه مع Prisma على المشروع نفسه في الدورة؟

## الخلاصة

TypeORM مسار رسمي بالكيان والمستودع. نقرأه حتى نفهم التوثيق، ونطفئ المزامنة التلقائية، ونُبقي بناء المكتبة على Prisma.

## ماذا بعد؟

[Drizzle و @nestjs/drizzle](./12-drizzle.md) البديل الموثّق الذي يبني الجداول بدوال TypeScript.

## المصادر الرسمية

- NestJS — TypeORM: https://docs.nestjs.com/data/typeorm
