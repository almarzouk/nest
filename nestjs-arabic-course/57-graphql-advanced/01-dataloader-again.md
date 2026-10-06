# DataLoader مرة أخرى

## الهدف من هذا الدرس

تجميع قراءات المؤلف في استعلام كتب رف داخل الطلب الواحد، باستخدام مكتبة DataLoader مع Nest، لا باستخدام ديكور يزعم أنه يحل N+1 وحده.

## المشكلة التي يحلها

حقل `author` على كل كتاب يستدعي `findUnique` من داخل `@ResolveField`. عشرون كتابًا تعني عشرين استعلامًا زيادة على استعلام القائمة. الفريق جرّب ذاكرة على خدمة singleton فخلط مؤلفين بين طلبات مستأجرين مختلفين.

## الفكرة ببساطة

خلال الطلب، اجمع معرّفات المؤلفين التي ستُطلب في نفس الدورة، ثم اقرأها باستعلام واحد، ووزّع الصفوف على الحقول. هذه مهمة DataLoader. Nest يوفّر مكان الحقن. المكتبة توفّر التجميع. لا يوجد `@DataLoader()` رسمي في `@nestjs/graphql` يغني عن الاثنين.

## التعريف التقني

`DataLoader` من حزمة `dataloader` تأخذ دالة `batch(keys)` تعيد مصفوفة بنفس ترتيب المفاتيح. تُنشأ داخل نطاق الطلب (`Scope.REQUEST`) أو داخل سياق GraphQL الذي يُبنى لكل عملية، حتى لا تخدم الذاكرة طلبًا آخر. في GraphQL، حقن الطلب يتم عبر `CONTEXT` لا عبر `REQUEST` كما في صفحة Injection scopes. الحقل يستدعي `loader.load(authorId)` ولا يستدعي Prisma مباشرة.

## Mental Model

```text
كتاب 1 load(a1) ─┐
كتاب 2 load(a1) ─┼─ دورة واحدة → WHERE id IN (a1, a2)
كتاب 3 load(a2) ─┘
```

## أبسط مثال

```ts
import DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { AuthorsService } from '../authors/authors.service.js';

@Injectable({ scope: Scope.REQUEST })
export class AuthorLoader {
  readonly loader: DataLoader<string, { id: string; name: string } | null>;

  constructor(authors: AuthorsService) {
    this.loader = new DataLoader(async (ids) => {
      const rows = await authors.findManyByIds([...ids]);
      const byId = new Map(rows.map((row) => [row.id, row]));
      return ids.map((id) => byId.get(id) ?? null);
    });
  }
}
```

## شرح المثال

- ترتيب الإرجاع يطابق ترتيب `ids`. خلط الترتيب يربط كتابًا بمؤلف آخر.
- `Scope.REQUEST` هنا مقصود: عمر المجمّع عمر العملية. كلفته نوقشت في فصل الأداء؛ لا ترفع كل الخدمات إلى هذا النطاق، ارفع المجمّع فقط.
- `findManyByIds` استعلام واحد. إن بقيت `findUnique` داخل `batch` فقد أعدت N+1 باسم جديد.

## ماذا يحدث داخليًا؟

1. GraphQL يحل حقول `author` في نفس المستوى.
2. كل `load` يسجّل المفتاح ولا يذهب إلى القاعدة فورًا.
3. في نهاية دورة الأحداث الحالية، DataLoader يستدعي `batch` مرة.
4. المفاتيح المكررة (نفس المؤلف على كتابين) تُستعلم مرة.
5. بعد انتهاء الطلب تموت النسخة الطلبية فلا تبقى نتائج مستأجر في طلب تالٍ.

## مثال عملي

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { AuthorLoader } from './author.loader.js';

@Resolver('Book')
export class BookAuthorResolver {
  constructor(private readonly authors: AuthorLoader) {}

  @ResolveField()
  author(@Parent() book: { authorId: string }) {
    return this.authors.loader.load(book.authorId);
  }
}
```

`@Resolver` و `@ResolveField` من `@nestjs/graphql`. التجميع من `dataloader`. إن نسيت تسجيل `AuthorLoader` كمزوّد في الوحدة، الحقل لا يجد مجمّعًا. هذا ليس فشل ديكور واحد ينقصه خيار.

## أخطاء شائعة

- مجمّع singleton يعيش طول العملية فيخلط الطلبات.
- `batch` تعيد الصفوف بترتيب القاعدة لا بترتيب المفاتيح.
- فتح DataLoader داخل كل `load` فلا يتجمع شيء.
- الاعتقاد أن `@nestjs/graphql` يجمّع علاقات Prisma تلقائيًا.

## متى نستخدمه؟

لحقل علاقة يُحل لكثير من الآباء في عملية واحدة: مؤلف الكتاب، عدد المراجعات، الناشر.

## متى لا نستخدمه؟

لاستعلام جذر يعيد كتابًا واحدًا بمعرّفه؛ الاستعلام الواحد أوضح. ولا ككاش عبر الطلبات. لذلك يوجد تخزين مؤقت بمفتاح مستأجر إن كان الناتج عامًا.

## مقارنة

| الأداة | دورها |
| --- | --- |
| `@ResolveField` | يعلن كيف يُحسب الحقل |
| `DataLoader` | يجمع المفاتيح في الطلب |
| `Scope.REQUEST` أو سياق العملية | يعزل الذاكرة |
| فهرس SQL | يسرّع `IN (...)` الناتج |

## تمارين

### تمرين سهل

عشرون كتابًا لعشرة مؤلفين. كم مرة تُستدعى `batch` إن شُغّلت الحقول في مستوى واحد؟

<details>
<summary>الحل</summary>

مرة واحدة في تلك الدورة، و `ids` فيها عشرة مفاتيح بعد إزالة التكرار داخل DataLoader، لا عشرون استعلام `findUnique`.

</details>

### تمرين متوسط

لماذا المجمّع العام على التطبيق خطر في رف متعدد المستأجرين؟

<details>
<summary>الحل</summary>

الذاكرة تبقى بعد الطلب. طلب مكتبة أخرى قد يحصل على مؤلف من ذاكرة الطلب السابق إن تطابق المعرّف أو إن نسي `batch` قيد `tenantId`. عمر المجمّع عمر العملية، و `batch` نفسه يقيّد المستأجر.

</details>

### تمرين متقدم

`batch` أعادت صفين بينما المفاتيح ثلاثة، وبترتيب مختلف. ماذا يحدث للحقل الثالث؟

<details>
<summary>الحل</summary>

DataLoader يربط بالنظر إلى الموضع. نقص عنصر أو اختلاف ترتيب يربط القيمة الخطأ أو يرمي. أعد مصفوفة بطول المفاتيح: القيمة أو `null` في موضع المفتاح الغائب.

</details>

## أسئلة مراجعة

- أي حزمة تجمع المفاتيح، وأي حزمة تعلن الحقل؟
- لماذا الترتيب في `batch` جزء من الصحة؟
- لماذا لا يكفي singleton؟
- هل Nest يقدّم ديكورًا واحدًا يغني عن DataLoader؟

## الخلاصة

DataLoader مجمّع لكل طلب تكتبه أنت حول استعلام `IN`. `@ResolveField` يستدعي `load`. لا ديكور رسمي واحد يلغي N+1 بالنيابة عن هذا التصميم.

## ماذا بعد؟

التجميع يمنع شكلًا واحدًا من الانفجار. الدرس التالي يسمّي هذا الشكل N+1 ويعرض من أين يأتي في المخطط.

## المصادر الرسمية

- DataLoader: https://github.com/graphql/dataloader
- NestJS — GraphQL resolvers: https://docs.nestjs.com/graphql/resolvers
- NestJS — Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
