# Mocking Providers

## الهدف من هذا الدرس

استبدال مزوّد مسجّل داخل وحدة مستوردة بـ `overrideProvider` قبل `compile`.

## المشكلة التي يحلها

`BooksModule` تسجل `BooksService` و `PrismaService` معًا. إعادة كتابة `providers` في الاختبار تنسخ الوحدة وتبتعد عنها. تمرير `useValue` بجانب الصنف نفسه يسجل نسختين إن لم تُزل الأصلية. `overrideProvider` يستبدل الرمز الذي ستحقنه الحاوية.

## الفكرة ببساطة

استورد الوحدة كما في التطبيق، ثم سلسلة `overrideProvider(Token).useValue(mock)` أو `useClass`، ثم `compile`. كل حقن لذلك الرمز يرى الوهم. الخدمة تحت الاختبار تبقى الصنف الحقيقي.

## التعريف التقني

```ts
const prisma = {
  book: {
    update: vi.fn(async () => ({ id: '42', status: 'PUBLISHED' })),
  },
};

const moduleRef = await Test.createTestingModule({
  imports: [BooksModule],
})
  .overrideProvider(PrismaService)
  .useValue(prisma)
  .compile();
```

على قالب Jest استبدل `vi.fn` بـ `jest.fn`. الفكرة واحدة.

`overrideProvider` للصنف أو للرمز مثل `CACHE_MANAGER` أو `getQueueToken(coverQueueName)` من `@nestjs/bullmq` إن كان الطابور مسجّلًا في الوحدة. وهم الطابور `{ add: async () => ({ id: 'job' }) }` يمنع Redis.

أشكال أخرى من الصفحة عند الحاجة: `overrideGuard` و `overrideInterceptor` و `overridePipe` و `overrideFilter`. لاستبدال وحدة كاملة `overrideModule().useModule(...)`. لا تستبدل الحارس في كل اختبار إن كان موضوع الرحلة هو الحارس نفسه.

## Mental Model

```text
imports BooksModule
  override Prisma و Queue
    compile
      BooksService الحقيقية تعمل على الوهم
```

## أبسط مثال

```ts
.overrideProvider(getQueueToken(coverQueueName))
.useValue({ add: async () => ({ id: 'job-1' }) })
```

بعد إنشاء كتاب تتوقع `add` استُدعي بـ `'resize'` وحمولة فيها `bookId` لا نموذج Prisma.

## شرح المثال

`getQueueToken` هو رمز `@InjectQueue`. تمرير السلسلة `'covers'` وحدها قد لا يطابق الرمز الداخلي. استعمل دالة الحزمة حتى يطابق الاستبدال ما تحقنه Nest. بعد `compile` أي خدمة في الوحدة تطلب الطابور تأخذ `add` الوهمي.

## ماذا يحدث داخليًا؟

الباني يبدّل تعريف المزود قبل إنشاء الحاوية. النسخة الواحدة من الوهم تذهب إلى كل من يحقن الرمز في هذه الوحدة الاختبارية، ما لم يكن المزود `request` النطاق. الأوهام كائنات في الذاكرة. `mockClear` أو إنشاء وحدة جديدة في `beforeEach` يمنع توقعًا من حالة سابقة. `useClass` يبني صنفًا بديلًا بالحاوية إن كان البديل يحتاج حقنًا أيضًا. `useValue` أبسط عندما لا يحتاج الوهم إلى شيء.

## مثال عملي

رحلة HTTP ما زالت تستبدل الطابور لا الخدمة:

```ts
const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
  .overrideProvider(PrismaService)
  .useValue(prismaTest)
  .overrideProvider(getQueueToken(coverQueueName))
  .useValue({ add: async () => ({ id: 'job-1' }) })
  .compile();
```

`AppModule` هنا ثقيل. مقبول لرحلة واحدة، لا لكل فرع. الأفضل وحدة `BooksModule` زائد ما ينقصها من إعداد. إن سحب الجذر `ScheduleModule` فقد تنطلق مؤقتات. استبدلها أو لا تستورد الجذر.

## أخطاء شائعة

- `overrideProvider` بعد `compile`. الاستبدال يُتجاهل.
- توقع أن الوهم يصل إلى وحدة لم تُستورد في هذا `createTestingModule`.
- استبدال `BooksService` ثم اختبار `BooksService`. استبدلت ما تريد إثباته.
- نسيان رمز الطابور واستبدال صنف `Queue` العام.

## متى نستخدمه؟

عندما تكون الوحدة الحقيقية هي موضوع الاختبار والحدود حولها تُستبدل: قاعدة، طابور، ناشر.

## متى لا نستخدمه؟

لاستبدال الصنف الذي تختبر فروعه. ولا كطريقة لتجنب استخراج دالة نقية كانت ستُختبر بـ `new`.

## مقارنة

| الأسلوب | الوحدة الأصلية |
| --- | --- |
| نسخ `providers` يدويًا | قد تنحرف |
| `overrideProvider` قبل `compile` | تبقى، والرمز يُستبدل |
| `useValue` في القائمة مع الصنف نفسه | نسختان إن لم يُزل الأصل |

## تمارين

### تمرين سهل

قبل أي نداء يجب أن يقع `overrideProvider`؟

<details>
<summary>الحل</summary>

قبل `compile`.

</details>

### تمرين متوسط

لماذا `getQueueToken` لا السلسلة فقط؟

<details>
<summary>الحل</summary>

لأن الحقن يستعمل الرمز الذي تبنيه الحزمة من الاسم. السلسلة قد لا تطابق ذلك الرمز.

</details>

### تمرين متقدم

ماذا يصبح الاختبار إن استبدلت `BooksService` في ملف يزعم اختبار النشر؟

<details>
<summary>الحل</summary>

يثبت أن الحاوية تعيد الوهم، لا أن `publish` يحفظ ويعلن. اترك الخدمة واستبدل Prisma والمُصدِر.

</details>

## أسئلة مراجعة

- أي مزودات تُستبدل حول الخدمة لا الخدمة نفسها؟
- متى `useClass` بدل `useValue`؟
- لماذا الجذر الكامل خطر على المؤقتات؟

## الخلاصة

`overrideProvider(token).useValue(mock)` قبل `compile` يبدّل حدًا داخل الوحدة الحقيقية. لا تستبدل الصنف الذي تريد مراقبة سلوكه.

## ماذا بعد؟

تطبيق ذلك على خدمة الكتب: حالات النشر بلا HTTP.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
