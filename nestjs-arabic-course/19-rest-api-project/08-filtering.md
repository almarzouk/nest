# Filtering

## الهدف من هذا الدرس

تضييق `GET /posts` بشرط اختياري يعرّفه العقد، مثل المقالات المنشورة أو مقالات مؤلف، من غير تمرير كائن `where` خام.

## المشكلة التي يحلها

قائمة الصفحات ما زالت تعيد المسودات مع المنشور. العميل الذي يريد مقالات مؤلف واحد يسحب الصفحات كلها ويرشّح في المتصفح. فتح `where` كـ JSON في الاستعلام يتيح للعميل اختيار أي عمود، ومنها حقول لاحقًا حساسة.

## الفكرة ببساطة

أضف حقولًا اختيارية إلى `ListPostsQueryDto`: `published` و `authorId`. الخدمة تبني `where` من هذه الحقول فقط. قيمة غائبة تعني «لا تضيّق هذا الشرط». قيمة حاضرة تُضاف. الصفحة تُحسب بعد التضييق، والعدد الكلي هو عدد المطابق لا عدد الجدول كله.

## التعريف التقني

```ts
@IsOptional()
@Type(() => Boolean)
@IsBoolean()
published?: boolean;

@IsOptional()
@Type(() => Number)
@IsInt()
@Min(1)
authorId?: number;
```

تحويل `published` من الاستعلام حساس: النص `"false"` قد يصبح حقيقة في تحويل ساذج لأن أي نص غير فارغ حقيقة في JavaScript. لا تعتمد على `enableImplicitConversion` لهذا الحقل. حوّله صراحة:

```ts
function parsePublished(value: unknown): boolean | undefined {
  if (value === undefined) return undefined;
  if (value === "true" || value === true) return true;
  if (value === "false" || value === false) return false;
  throw new BadRequestException("published must be true or false");
}
```

أو استخدم أنبوبًا صغيرًا على المعامل. المهم أن `"false"` لا يمر كـ `true`.

في الخدمة:

```ts
const where = {
  ...(query.published !== undefined ? { published: query.published } : {}),
  ...(query.authorId !== undefined ? { authorId: query.authorId } : {}),
};
```

ثم `findMany({ where, skip, take, orderBy })` و `count({ where })`.

لا تبنِ تصفية عامة بكل أعمدة `Post`. الفصل `26` يناقش قائمة الأعمدة المسموحة كعقد قابل لإعادة الاستخدام.

## Mental Model

```text
?published=true&authorId=4&page=1
        |
 where { published: true, authorId: 4 }
        |
 صفحة من المطابق فقط
```

## أبسط مثال

```http
GET /posts?published=true&authorId=4&page=1&pageSize=20 HTTP/1.1
```

المسودات لا تدخل `items` ولا `total`. مقالات المؤلف `5` لا تدخل. الصفحة `1` أول عشرين منشورًا مطابقًا حسب الترتيب الحالي `id`.

## شرح المثال

الشرطان معًا يعنيان AND، وهو ما يفعله كائن `where` في Prisma عند وضع الحقول جنبًا إلى جنب. إن غاب `authorId` يبقى شرط النشر فقط. إن غاب الاثنان تعود القائمة كما في درس الترقيم.

مؤلف غير موجود يعيد صفحة فارغة و `total: 0`، لا `404`. المسار قائمة مرشّحة، والفلتر الذي لا يطابق شيئًا نتيجة فارغة صحيحة. `404` يبقى لـ `GET /posts/:id`.

## ماذا يحدث داخليًا؟

Prisma يضيف `WHERE` بمعاملات مربوطة. اسم العمود ثابت في الكود لأنك كتبته `published` لا من نص العميل. هذا الفرق بين فلتر مسموح و SQL يُبنى من اسم عمود يرسله الطلب.

فهرس على `(authorId, published)` يجعل الشرط رخيصًا حين يكبر الجدول. ليس مطلوبًا للتمرين، ويُذكر الآن حتى لا تُفاجأ في الفصل `26`.

## مثال عملي

انشر مقالين للمؤلف `4` واترك ثالثًا مسودة. `published=true` يعيد اثنين. `published=false&authorId=4` يعيد المسودة. `published=maybe` يعيد `400`. أرسل `passwordHash=x` في الاستعلام: الحقل ليس في DTO، و `forbidNonWhitelisted` يرفضه إن فُعّل على الاستعلام أيضًا. الأنبوب العالمي يفعل ذلك إن كان `validateCustomDecorators` ليس شرطًا لـ `@Query` على صنف. تأكد بالتجربة أن الزائد مرفوض.

## أخطاء شائعة

- اعتبار النص `"false"` قيمة منطقية كاذبة عبر `Boolean("false")`، وهي `true`.
- `count()` بلا `where` مع `findMany` مضيّق، فيكذب شريط الصفحات.
- تمرير `req.query` إلى Prisma.
- `404` عندما لا توجد نتائج.

## متى نستخدمه؟

حين للعميل سؤال متكرر ضيّق: منشور، أو مؤلف، أو الاثنان.

## متى لا نستخدمه؟

لبحث نصي حر في `body` قبل أن تقرر لغة البحث. `contains` ممكن لاحقًا كحقل مسمّى `q`، ليس في هذا الدرس.

## مقارنة

| فلتر في المتصفح | `where` من DTO |
| --- | --- |
| يسحب كل الصفحات | يضيّق في القاعدة |
| يرى المسودات إن أرسلتها | يقدر أن يخفيها |
| لا فهرس يساعده | الشرط عمود حقيقي |

## تمارين

### تمرين سهل

ماذا تعني غياب `published` في الاستعلام؟

<details>
<summary>الحل</summary>

لا تضف حقل `published` إلى `where`. المسودات والمنشورات كلاهما مؤهّل إن لم يوجد شرط آخر.

</details>

### تمرين متوسط

لماذا `Boolean("false")` لا يصلح هنا؟

<details>
<summary>الحل</summary>

أي نص غير فارغ في JavaScript يتحوّل إلى `true`، ومنها `"false"`. الحل مقارنة صريحة بالنصين `true` و `false`.

</details>

### تمرين متقدم

العميل يريد استبعاد مؤلف. لماذا لا نضيف `authorIdNot` في اليوم نفسه؟

<details>
<summary>الحل</summary>

كل عامل جديد عقد. ابدأ بالمساواة التي يحتاجها المنتج. النفي والقوائم يأتيان حين يتكرر الطلب، ويُسمّيان في DTO لا ككائن حر. الفصل `26` يوسّع الأنماط بعد ثبات الحاجة.

</details>

## أسئلة مراجعة

- كيف يُجمع شرطان في `where`؟
- لماذا `total` يستخدم الشرط نفسه؟
- ماذا يرجع فلتر بلا نتائج؟

## الخلاصة

التصفية حقول اختيارية مسمّاة تُترجم إلى `where` ثابت الأعمدة. الصفحة والعدد يتبعان الشرط. اسم العمود لا يأتي من العميل.

## ماذا بعد؟

[Sorting](./09-sorting.md) يسمح بترتيب القائمة بحقول من قائمة بيضاء.

## المصادر الرسمية

- NestJS — Validation: https://docs.nestjs.com/application/validation
- NestJS — Prisma: https://docs.nestjs.com/data/prisma
