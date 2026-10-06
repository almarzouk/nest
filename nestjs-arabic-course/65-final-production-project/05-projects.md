# Projects

## الهدف من هذا الدرس

إضافة مورد المشروع في رف: كتاب قيد التجهيز داخل المنظمة، بقائمة 200 إن كانت فارغة وإنشاء 201، وكل استعلام مقيّد بـ `organizationId`.

## المشكلة التي يحلها

الكتب تُعلَّق على المستخدم. أو `GET /projects` بلا صفوف يعيد 404 فتظن الواجهة أن المسار خطأ. أو `POST` يعيد 200. أو `findUnique` على `id` فقط فيفتح كتاب الجار.

## الفكرة ببساطة

`Project` له `title` و `status` يبدأ `draft` و `organizationId` من السياق. القائمة غلاف `{ data, nextCursor }`. الإنشاء 201. الفرد الغائب داخل المكتبة 404. النشر إلى `published` فعل لاحق يمكنه العيش هنا كـ `POST /projects/:id/publish` بشرط أن المالك فقط هو الموجود الآن؛ الخطة تُضاف في الدرس التالي فلا تمنعوا النشر قبلها إلا بتعليق في الكود يوضح أن الحارس قادم.

## التعريف التقني

```prisma
model Project {
  id             String @id
  organizationId String
  title          String
  status         String
  @@index([organizationId, status])
}
```

`POST /projects` لا يقبل `organizationId` في الجسم؛ يُكتب من السياق. `GET /projects` 200 و `data: []`. `@HttpCode(201)` على الإنشاء. التحديث `PATCH` على حقول العنوان فقط ما دام `draft`. لا GraphQL في هذه الشريحة.

## Mental Model

```text
سياق المنظمة
→ WHERE organizationId
→ draft يُنشأ
→ القائمة [] ما زالت 200
```

## أبسط مثال

```ts
@Get()
list() {
  return this.projects.list(currentTenantId());
}
```

الخدمة تعيد `{ data: rows, nextCursor: null }` حتى لو `rows` فارغة. لا `NotFoundException`.

## شرح المثال

- المتحكم لا يمرر جسم المستأجر.
- `list` تضع `take: 20`.
- الفهرس يخدم هذا `where`.

## ماذا يحدث داخليًا؟

1. `TenantGuard` يملأ السياق.
2. `POST` يدرج `draft` و `organizationId` ويعيد 201 مع `id`.
3. `GET /projects/:id` يستخدم `findFirst({ where: { id, organizationId } })`.
4. غياب الصف 404 `project_not_found` بشكل الخطأ الموحّد.
5. منظمة أخرى لا ترى الصف حتى لو عرفت `id`.

## مثال عملي

```ts
@Post()
@HttpCode(201)
create(@Body() body: { title: string }) {
  return this.projects.create(currentTenantId(), body.title);
}
```

```ts
create(organizationId: string, title: string) {
  return this.prisma.project.create({
    data: { id: randomUUID(), organizationId, title, status: 'draft' },
  });
}
```

`randomUUID` من `node:crypto`. لا حالة `published` من جسم الإنشاء. النشر مسار منفصل حتى يُربَط بالخطة لاحقًا.

## أخطاء شائعة

- 404 للقائمة الفارغة.
- 200 على الإنشاء.
- `status` من العميل عند `POST` فيُنشر بلا سياسة.
- كاش قائمة بلا `organizationId`.

## متى نستخدمه؟

عندما توجد منظمة. هذا قلب المنتج في الفصل.

## متى لا نستخدمه؟

لا تضيفوا فصول الكتاب والمراجعات. المشروع صف واحد في هذا النطاق. ولا تمنعوا الإنشاء بـ Stripe قبل درس الاشتراك؛ يمكنكم السماح بمسودات بلا خطة ثم قفل النشر.

## مقارنة

| الطلب | رمز |
| --- | --- |
| قائمة فارغة | 200 و `data: []` |
| إنشاء | 201 و `draft` |
| معرّف غائب في المكتبة | 404 |
| معرّف في مكتبة أخرى | 404 لنفس المستدعي |

## تمارين

### تمرين سهل

ماذا يعيد `GET /projects` لمكتبة جديدة؟

<details>
<summary>الحل</summary>

200 و `{ data: [], nextCursor: null }` لا 404.

</details>

### تمرين متوسط

لماذا `status` ليس في جسم الإنشاء؟

<details>
<summary>الحل</summary>

حتى لا يرسل العميل `published` ويتجاوز الخطة. البداية `draft` دائمًا. النشر مسار لاحق.

</details>

### تمرين متقدم

اختبار يعرض `id` مشروع دار النيل لترويسة مكتبة أخرى. ماذا تتوقعون؟

<details>
<summary>الحل</summary>

404. `where` فيه منظمة الترويسة. الصف لا يُعدَّل ولا يُكشف عنوانه.

</details>

## أسئلة مراجعة

- ماذا يمثّل `Project`؟
- من أين `organizationId`؟
- ما رمز القائمة الفارغة والإنشاء؟
- لماذا لا `findUnique` على المعرّف وحده؟

## الخلاصة

المشروع كتاب `draft` داخل المنظمة. القائمة الفارغة 200 والإنشاء 201. العزل شرط على كل استعلام، والنشر العام لم يُربط بالخطة بعد.

## ماذا بعد؟

النشر يحتاج اشتراكًا لا يُفعَّل من صفحة النجاح. الدرس التالي الدفع.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
- MDN — 201: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/201
