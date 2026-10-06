# Jest و Vitest

## الهدف من هذا الدرس

اختيار مشغّل الاختبار الذي يطابق قالب المشروع: Vitest لقالب ESM من `nest new`، و Jest لقالب CommonJS.

## المشكلة التي يحلها

نسخ `jest.fn` في مشروع يشغّل Vitest يفشل لأن الاسم غير معرّف، أو العكس. صفحة Nest للاختبار تشرح `TestingModule` وهي واحدة. المشغّل الذي ينفّذ الملف يعتمد على القالب. خلط الإعدادين يعطي أخطاء تحويل TypeScript وهمية تبدو كأخطاء Nest.

## الفكرة ببساطة

ESM الحديث مع `nest new` الافتراضي يستخدم Vitest. قالب CommonJS يستخدم Jest. `expect` و `describe` و `it` متشابهان. الفرق في الاستيراد وفي اسم دالة الوهم وفي ملف الإعداد. لا تثبت المشغّلين معًا على التطبيق نفسه.

## التعريف التقني

من سلوك القوالب الحالية:

| القالب | المشغّل | الوهم |
| --- | --- | --- |
| `nest new` على ESM | Vitest | `vi.fn()` بعد `import { vi } from 'vitest'` |
| قالب CommonJS | Jest | `jest.fn()` والدوال عامة غالبًا |

Vitest:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest';

describe('BooksService', () => {
  it('announces publish', async () => {
    const emitAsync = vi.fn(async () => []);
    const service = new BooksService(prisma, { emitAsync } as never);
    await service.publish('42');
    expect(emitAsync).toHaveBeenCalledWith(bookPublished, { bookId: '42' });
  });
});
```

Jest على القالب الآخر:

```ts
describe('BooksService', () => {
  it('announces publish', async () => {
    const emitAsync = jest.fn(async () => []);
    expect(emitAsync).toHaveBeenCalled();
  });
});
```

لا تنسخ كتلة Jest إلى مشروع Vitest. إن كان الدرس يعرض السلوك، فالوهم يُكتب بأداة القالب الذي أنشأته.

`TestingModule` من `@nestjs/testing` يعمل مع الاثنين. لا يوجد `TestingModule` خاص بـ Vitest.

## Mental Model

```text
قالب ESM -> vitest
قالب CommonJS -> jest
@nestjs/testing -> نفسه
```

## أبسط مثال

أمر التشغيل من سكربت الحزمة، لا من ذاكرة المقال:

```text
npm test
```

السكربت في قالب ESM يستدعي Vitest، وفي قالب CommonJS يستدعي Jest. إن فشل الأمر لأن `jest` غير موجود، فأنت على القالب الأول. لا تثبت Jest «لإصلاح» الرسالة قبل أن تقرأ `package.json`.

## شرح المثال

تثبيت المشغّل الثاني يخلق ملفين إعداد وسلوكين للاستيراد `.js`. الدورة تستعمل ESM في الأمثلة، فالأمثلة التي تحتاج وهمًا تكتب `vi` عندما يكون المشغّل ظاهرًا. الدروس التي تركز على Nest تعرض `expect` فقط حيث لا يهم اسم الوهم.

## ماذا يحدث داخليًا؟

المشغّل يحوّل TypeScript ويشغّل الملفات المطابقة لنمط `*.spec.ts`. زينة Nest تحتاج بيانات وصفية للتصميم. قوالب Nest تضبط التحويل لذلك. إن استبدلت الإعداد بقالب Vite فارغ، قد يصبح الحقن `undefined` لأن المعاملات بلا نوع وقت التشغيل. عندها أصلح إعداد القالب الرسمي لا تكتب `@Inject()` على كل باني كحل دائم.

العزل: كل ملف عملية أو بيئة حسب المشغّل. لا تعتمد على متغير عُيّن في ملف آخر.

## مثال عملي

مشروع الدورة ESM:

```ts
import { vi } from 'vitest';

const covers = { enqueueResize: vi.fn(async () => undefined) };
```

مشروع قديم CommonJS في المستودع نفسه لا تخلطه. إن نقلت ملفًا بينهما، بدّل `vi.fn` و `jest.fn` مع الإعداد لا السطر فقط.

## أخطاء شائعة

- `jest.fn` في قالب Vitest.
- تثبيت Jest و Vitest معًا بلا سبب.
- افتراض أن فشل الحقن خطأ `TestingModule` بينما التحويل أسقط الأنواع.
- تشغيل `npm test` من مجلد ليس جذر القالب.

## متى نستخدمه؟

Vitest مع ما يولّده `nest new` على ESM اليوم. Jest إن أنشأت القالب CommonJS أو ورثت مشروعًا عليه.

## متى لا نستخدمه؟

لا تبدّل المشغّل في منتصف الفصل لتجربة مقال. `Test.createTestingModule` هو موضوع الدرس التالي على المشغّل الذي عندك.

## مقارنة

| | Vitest | Jest |
| --- | --- | --- |
| قالب Nest الحالي ESM | الافتراضي | لا تفرضه |
| قالب CommonJS | ليس افتراضه | الافتراضي |
| الوهم | `vi.fn` | `jest.fn` |
| وحدة الاختبار Nest | `@nestjs/testing` | `@nestjs/testing` |

## تمارين

### تمرين سهل

أي مشغّل مع `nest new` على ESM؟

<details>
<summary>الحل</summary>

Vitest.

</details>

### تمرين متوسط

ماذا تفعل إن نُسخ `jest.fn` إلى مشروع Vitest؟

<details>
<summary>الحل</summary>

استبدله بـ `vi.fn` مع استيراد `vi` من `vitest`. لا تثبت Jest بجانب القالب.

</details>

### تمرين متقدم

لماذا قد يفشل حقن الباني بعد تغيير إعداد التحويل؟

<details>
<summary>الحل</summary>

لأن Nest تقرأ أنواع المعاملات من البيانات الوصفية. إعداد يتجاهلها يجعل الاعتماد `undefined`. أرجع إعداد قالب Nest لا زينة `@Inject` على كل معامل.

</details>

## أسئلة مراجعة

- أي حزمة اختبار Nest مشتركة؟
- أي قالب يبقي Jest؟
- أي أمر تشغّل من الجذر؟

## الخلاصة

المشغّل يتبع القالب: Vitest لـ ESM و Jest لـ CommonJS. `expect` متشابه و `TestingModule` واحد. لا تثبت الاثنين معًا.

## ماذا بعد؟

بناء حاوية اختبار بـ `Test.createTestingModule` واستخراج الصنف منها.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
