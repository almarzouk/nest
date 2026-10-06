# شكل ذلك داخل NestJS

## الهدف من هذا الدرس

تركيب عزل رف داخل Nest: حارس يحل المكتبة، سياق يعيش مع الطلب، وخدمة singleton تستقبل `tenantId` أو تقرأه من المخزن، مع استعلام مقيّد دائمًا.

## المشكلة التي يحلها

كل خدمة تحقن `REQUEST` لتقرأ الترويسة، فتصبح الشجرة طلبية كما في درس كلفة النطاق. ومسار جديد يستدعي `prisma.book.findMany()` بلا وسيط فينسى العزل. لا مكان واحد يراجع فيه الفريق الشكل.

## الفكرة ببساطة

الحارس أو الـ interceptor يحل المستأجر مرة ويضعه في `AsyncLocalStorage` أو في خاصية الطلب. الخدمات تبقى `DEFAULT`. دالة مستودع واحدة تغلف Prisma وتفرض `tenantId`. نموذج pool يضيف `where`. نموذج silo يختار العميل من الخريطة. الديكورات لا تعزل وحدها.

## التعريف التقني

`@Injectable()` بلا `Scope.REQUEST` للخدمات العادية. المخزن من `node:async_hooks` يُفتح حول تنفيذ الطلب. إن احتجتم شجرة DI لكل مستأجر وعدد المكتبات صغير، المزودات الدائمة في صفحة Injection scopes خيار موثّق لا الخطوة الأولى. `APP_GUARD` يشغّل الحل بعد المصادقة حتى يوجد `user`. الدالة العامة `books.list(tenantId)` تجعل نسيان المعرّف خطأ TypeScript. لا تقرأوا `@Body('tenantId')` في المتحكم. الاختبار يستدعي الخدمة بمعرّفين ويتوقع العزل من غير HTTP إن أمكن، ثم e2e يثبت أن الجسم المخالف لا يغيّر السياق.

## Mental Model

```text
Guard: resolve + membership → shelfStorage.run({ tenantId })
Controller → BooksService.list()
BooksService يقرأ المخزن ويضعه في where أو يختار عميل القاعدة
```

## أبسط مثال

```ts
import { AsyncLocalStorage } from 'node:async_hooks';

export const tenantStorage = new AsyncLocalStorage<{ tenantId: string }>();

export function currentTenantId() {
  const store = tenantStorage.getStore();
  if (!store) throw new Error('tenant context missing');
  return store.tenantId;
}
```

## شرح المثال

- الغياب استثناء. الصمت يعيد كل الصفوف.
- الحارس هو من يستدعي `run`. خدمة تعمل في مهمة بلا `run` تفشل فورًا فتربطون المهمة بمستأجر صريح.
- المخزن يتبع السلسلة async داخل نفس الطلب.

## ماذا يحدث داخليًا؟

1. المصادقة تملأ المستخدم.
2. الحارس يحل المستأجر ويفحص العضوية ويرفض الجسم المخالف.
3. `tenantStorage.run({ tenantId }, callback)` يغلّف بقية السلسلة.
4. المتحكم يستدعي الخدمة بلا تمرير المكتبة من `@Body`.
5. المستودع يضع `currentTenantId()` في كل استعلام pool، أو يختار عميل silo.

## مثال عملي

```ts
import { CanActivate, ExecutionContext, Injectable, BadRequestException, ForbiddenException } from '@nestjs/common';
import { tenantStorage } from './tenant.storage.js';

@Injectable()
export class TenantGuard implements CanActivate {
  canActivate(context: ExecutionContext) {
    const req = context.switchToHttp().getRequest<{ user?: { memberships: { tenantId: string }[] }; headers: Record<string, string>; body?: { tenantId?: string } }>();
    const tenantId = req.headers['x-tenant-id'];
    if (!tenantId) throw new ForbiddenException('tenant missing');
    if (req.body?.tenantId && req.body.tenantId !== tenantId) throw new BadRequestException('tenant in body is ignored source');
    if (!req.user?.memberships.some((item) => item.tenantId === tenantId)) throw new ForbiddenException('not a member');
    tenantStorage.enterWith({ tenantId });
    return true;
  }
}
```

`enterWith` مناسب إن بقي التنفيذ على نفس السلسلة التي يكملها Nest بعد الحارس. إن انكسر السياق في اختبار، غلّفوا بالـ interceptor و `run` حول `next.handle()`. لا تعتمدوا على الجسم لملء الفراغ عندما تغيب الترويسة.

## أخطاء شائعة

- `Scope.REQUEST` على `PrismaService` لأن «كل مكتبة اتصال» في نموذج pool. اتصال واحد مشترك والعمود يكفي.
- قراءة الترويسة داخل الخدمة فتتعدد مصادر الحل.
- مهمة طابور تستدعي `currentTenantId()` بلا الدخول إلى `run` فتسقط، أو أسوأ: ترث سياق طلب سابق إن أسأتم استخدام `enterWith` خارج الطلب.
- كاش بمفتاح المسار فقط.

## متى نستخدمه؟

كالشكل القياسي لرف: حارس واحد، مخزن، مستودع مقيّد، خدمات singleton.

## متى لا نستخدمه؟

لا تبنوا `TenantModule` ديناميكيًا لكل طلب. ولا تستبدلوا الحارس بفلتر استثناءات؛ الفلتر يترجم الأخطاء ولا يختار المكتبة.

## مقارنة

| القطعة | الدور |
| --- | --- |
| Guard | حل + عضوية + رفض الجسم |
| AsyncLocalStorage | إيصال المعرّف بلا REQUEST على كل الشجرة |
| مستودع | `where` أو عميل القاعدة |
| Durable providers | بديل عندما مصادر قليلة وشجرة DI تعتمد عليها |

## تمارين

### تمرين سهل

لماذا تبقى `BooksService` بلا `Scope.REQUEST` في نموذج العمود؟

<details>
<summary>الحل</summary>

لأن لا حالة فيها خاصة بالطلب سوى المعرّف، وهو في المخزن. النطاق الطلبي يعيد بناء الشجرة بلا عزل إضافي.

</details>

### تمرين متوسط

مهمة ترحيل ليلية تستدعي `list()`. ماذا يحدث مع `currentTenantId()`، وما التعديل؟

<details>
<summary>الحل</summary>

ترمي `tenant context missing` وهذا أسلم من قراءة كل المكتبات. شغّلوا `tenantStorage.run({ tenantId }, () => list())` لكل مكتبة في الحلقة، أو مرروا المعرّف وسيطًا صريحًا في دالة المهمة.

</details>

### تمرين متقدم

ارسم اختبارين: وحدة للمستودع، و e2e للجسم المخالف.

<details>
<summary>الحل</summary>

الوحدة تستدعي `find(tenantA, idOfB)` وتتوقع `null`. الـ e2e يرسل ترويسة `lib-a` وجسم `tenantId: lib-b` ويتوقع 400، ثم طلبًا صحيحًا ويتوقع كتب `lib-a` فقط. فشل الثاني يعني أن المتحكم قرأ الجسم.

</details>

## أسئلة مراجعة

- أين يُحل المستأجر مرة واحدة؟
- كيف يصل المعرّف إلى خدمة singleton؟
- ماذا يرفض الحارس في الجسم؟
- متى تُذكر المزودات الدائمة؟

## الخلاصة

شكل Nest لعزل رف: حارس يملأ سياق الطلب بعد العضوية، خدمات singleton، ومستودع لا ينفّذ استعلامًا بلا `tenantId` أو بلا عميل تلك القاعدة. الجسم لا يملأ السياق.

## ماذا بعد؟

المكتبة معزولة. الفصل التالي يرسم ما حولها في منتج SaaS: عضوية وأدوار واشتراك وسجل تدقيق.

## المصادر الرسمية

- NestJS — Guards: https://docs.nestjs.com/guards
- NestJS — Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
- NestJS — Async local storage: https://docs.nestjs.com/recipes/async-local-storage
