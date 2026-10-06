# TestingModule

## الهدف من هذا الدرس

بناء وحدة Nest في الاختبار بـ `Test.createTestingModule` ثم أخذ نسخة الصنف بعد `compile`.

## المشكلة التي يحلها

`new BooksService` يكفي عندما تعرف الباني. عندما يأتي الاعتماد من وحدة مستوردة أو من رمز `CACHE_MANAGER`، البناء اليدوي يعيد تنفيذ التسجيل. الحاوية الاختبارية تقرأ `providers` و `imports` كما في التطبيق وتسمح باستبدال مزوّد قبل `compile`.

## الفكرة ببساطة

`Test.createTestingModule({ imports أو providers })` يعيد بانٍ. `compile()` يعيد `TestingModule`. `get(BooksService)` النسخة. أغلق الوحدة في النهاية إن كانت تحمل وصلات: `module.close()`.

## التعريف التقني

من صفحة الاختبار:

```ts
import { Test } from '@nestjs/testing';
import { BooksService } from './books.service.js';
import { BooksStore } from './books.store.js';

const moduleRef = await Test.createTestingModule({
  providers: [
    BooksService,
    { provide: BooksStore, useValue: { update: async () => ({ id: '42' }) } },
  ],
}).compile();

const service = moduleRef.get(BooksService);
await service.publish('42');
await moduleRef.close();
```

`useValue` مزوّد بديل بالرمز `BooksStore`. الدرس التالي يركز على `overrideProvider` عندما يكون الصنف مسجّلًا داخل `imports: [BooksModule]` ولا تريد نسخ القائمة.

`get` يرمي إن لم يُسجل الرمز. هذا أفضل من `undefined` صامت.

لا تستورد `AppModule` كاملًا في وحدة خدمة إن كان يسحب Redis وجدول الإنتاج. استورد وحدة الميزة أو صرّح المزودات التي تحتاجها.

## Mental Model

```text
createTestingModule
  compile
    get(BooksService)
      close
```

## أبسط مثال

```ts
const moduleRef = await Test.createTestingModule({
  providers: [BooksService, { provide: EventEmitter2, useValue: { emitAsync: async () => [] } }],
}).compile();
```

بعد `compile` الخدمة موجودة و `EventEmitter2` هو الوهم. نداء `publish` يسلك الكود الحقيقي للخدمة.

## شرح المثال

الترتيب: ابنِ، `compile`، `get`، نفّذ، `close`. `get` قبل `compile` غير متاح. إنشاء الوحدة داخل كل `it` يعزل الحالة. إن وضعتها في `beforeAll` فشارك الأوهام بين الحالات ما لم تُعد ضبطها في `beforeEach`.

## ماذا يحدث داخليًا؟

Nest تبني حاوية صغيرة وتقرأ الزينة على `BooksService` ليحقن الرموز. إن سقطت أنواع المعاملات عند التحويل، `get` يعيد خدمة فيها `undefined`. هذا موضوع إعداد المشغّل لا نقص `providers`. المزود `useValue` لا يمر على باني الصنف الأصلي، لذلك أثر `OnModuleInit` في المخزن الحقيقي لن يعمل، وهذا مطلوب في الوحدة.

`createNestApplication()` من الوحدة نفسها عندما تنتقل إلى HTTP. للوحدة التي تختبر الخدمة فقط لا تبنِ التطبيق.

## مثال عملي

مخبأ في حاوية:

```ts
const cache = { get: async () => undefined, set: async () => undefined, del: async () => undefined };
const moduleRef = await Test.createTestingModule({
  providers: [IsbnService, { provide: CACHE_MANAGER, useValue: cache }, httpDouble],
}).compile();
```

`CACHE_MANAGER` رمز وليس صنفًا. `useValue` بالمفتاح نفسه الذي تحقنه الخدمة. بعد `lookup` الناجح تتوقع `cache.set` استُدعي بمفتاح `isbn:`.

## أخطاء شائعة

- استيراد `AppModule` في كل ملف وحدة.
- نسيان `compile`.
- نسيان `close` عندما يوجد مؤقت أو مقبس.
- `get` لصنف غير مسجّل ثم معالجة `undefined` في الاختبار.

## متى نستخدمه؟

عندما يكون الحقن من رموز أو من وحدة، أو عندما ستستبدل مزوّدًا مسجّلًا. الدالة النقية لا تحتاج الحاوية.

## متى لا نستخدمه؟

لتشغيل التطبيق كاملًا مع Redis الحقيقي في كل `it`. الحاوية للاختبار، والوصلات الحقيقية تُحصر في تكامل قليل.

## مقارنة

| البناء | متى |
| --- | --- |
| `new Service(dep)` | الباني واضح ونقي |
| `createTestingModule` | رموز ووحدات وزينة |
| `createNestApplication` | رحلة HTTP |

## تمارين

### تمرين سهل

أي دالة تعيد الخدمة من الحاوية؟

<details>
<summary>الحل</summary>

`moduleRef.get(BooksService)` بعد `compile`.

</details>

### تمرين متوسط

لماذا لا تستورد `AppModule` في اختبار `publish`؟

<details>
<summary>الحل</summary>

لأنها تسحب طوابير ومخزنًا وإعدادًا لا يحتاجها إثبات الإعلان. ركّب مزودات الخدمة فقط.

</details>

### تمرين متقدم

ماذا يحدث لـ `OnModuleInit` على صنف استُبدل بـ `useValue`؟

<details>
<summary>الحل</summary>

لا يعمل على الوهم. الوهم ليس نسخة الصنف. إن كنت تختبر الخطاف فسجّل الصنف الحقيقي لا `useValue`.

</details>

## أسئلة مراجعة

- ما ترتيب `compile` و `get`؟
- متى `close`؟
- أي رمز يُمرر لمدير المخبأ؟

## الخلاصة

`createTestingModule` ثم `compile` ثم `get` يبني الخدمة كما تسجلها، مع بدائل `useValue`. أغلق الوحدة إن حملت وصلات، ولا تسحب الجذر كله.

## ماذا بعد؟

`overrideProvider` عندما الصنف آتٍ من `imports` ولا تريد إعادة إعلانه.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
