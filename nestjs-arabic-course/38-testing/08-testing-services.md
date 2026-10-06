# اختبار Services

## الهدف من هذا الدرس

كتابة حالات `BooksService` و `IsbnService` بالحاوية أو بـ `new`، مع توقع الأثر لا توقع السجل.

## المشكلة التي يحلها

خدمة بلا اختبار تعود إلى استيراد SMTP. خدمة باختبار يطابق النص الداخلي سطرًا بسطر تنكسر عند إعادة التسمية. الاختبار الجيد يثبت العقد: حالة الصف، الإعلان، خريطة الناشر، وعدم `set` على المخبأ عند المهلة.

## الفكرة ببساطة

جهّز أوهامًا تسجّل النداءات. نفّذ عملية واحدة. افحص الخرج والنداء. حالة لكل فرع عقد لا لكل سطر. لا HTTP في هذا الملف.

## التعريف التقني

نشر ناجح على قالب ESM:

```ts
import { describe, expect, it, vi } from 'vitest';

it('emits after a successful update', async () => {
  const update = vi.fn(async () => ({ id: '42', status: 'PUBLISHED' }));
  const emitAsync = vi.fn(async () => []);
  const service = new BooksService(
    { book: { update } } as never,
    { emitAsync } as never,
  );
  await service.publish('42');
  expect(update).toHaveBeenCalledWith({
    where: { id: '42' },
    data: { status: 'PUBLISHED' },
  });
  expect(emitAsync).toHaveBeenCalledWith(bookPublished, { bookId: '42' });
});
```

قالب Jest يستعمل `jest.fn` والحزم العامة `describe` و `it` و `expect`.

فرع الفشل:

```ts
it('does not emit when update throws', async () => {
  const update = vi.fn(async () => {
    throw new Error('db down');
  });
  const emitAsync = vi.fn();
  const service = new BooksService({ book: { update } } as never, { emitAsync } as never);
  await expect(service.publish('42')).rejects.toThrow('db down');
  expect(emitAsync).not.toHaveBeenCalled();
});
```

`IsbnService`: وهم `cache.get` يعيد `undefined`، ووهم HTTP يرمي `AxiosError` مع `code: 'ECONNABORTED'`. التوقع: الوعد يرفض بـ `GatewayTimeoutException` و `cache.set` لم يُستدعَ.

## Mental Model

```text
حالة = ترتيب + فرع واحد
  توقع أثر ظاهر
```

## أبسط مثال

إصابة مخبأ:

```ts
const cache = {
  get: vi.fn(async () => ({ title: 'NestJS' })),
  set: vi.fn(),
};
```

بعد `lookup('978')` الجسم `{ title: 'NestJS' }` و `set` غير مستدعى لأن القيمة ليست `undefined`. هذا يثبت شرط `!== undefined` الذي يميز `null` المخزّن لاحقًا في حالة ثانية.

## شرح المثال

حالة `null` المخزّن: `get` يعيد `null`، والنتيجة `null`، وبلا شبكة. إن استدعيت الشبكة فقد كُسر الشرط. حالتان أوضح من اختبار واحد فيه عدة `if`.

## ماذا يحدث داخليًا؟

`vi.fn` يسجّل الوسائط. `toHaveBeenCalledWith` يقارنها. إن مررت كائن Prisma حقيقيًا بالخطأ فالاختبار بطيء ويهشم العزل. الوهم لا يتصل. `rejects.toThrow` ينتظر وعدًا مرفوضًا. نسيان `await` على `expect(...).rejects` يعطي نجاحًا كاذبًا. دائمًا `await` لتوقع الرفض.

## مثال عملي

جدول ملف `books.service.spec.ts`:

| الحالة | إثبات |
| --- | --- |
| نشر ناجح | `update` ثم `emitAsync` |
| فشل التحديث | لا إعلان |
| إنشاء مع غلاف | `add` بحمولة `{ bookId, coverKey }` بلا مفاتيح نموذج كاملة |

حمولة `add` تُفحص بالمفاتيح لا بمرجع الكائن. `expect(payload).toEqual({ bookId: '42', coverKey: 'a.png' })` يكفي. وجود `coverKey` وغياب دوال Prisma هو العقد.

## أخطاء شائعة

- `await` ناقص على `rejects`.
- حالة واحدة سعيدة فقط.
- توقع `logger.log` كنتيجة وحيدة.
- تشغيل الخدمة على `DATABASE_URL` المطور.

## متى نستخدمه؟

لكل خدمة فيها فرع يغيّر أثرًا: إعلان، مخبأ، طابور، خريطة خطأ.

## متى لا نستخدمه؟

لاختبار شكل JSON للمستخدم. ذلك المتحكم أو الرحلة. الخدمة تعيد كائن مجال أو ترمي استثناء Nest يمكن توقعه بنوعه: `toBeInstanceOf(BadGatewayException)`.

## مقارنة

| العقد | ماذا تتوقع |
| --- | --- |
| نشر | نداء تحديث وإعلان |
| مهلة ISBN | رمي `504` وبلا `set` |
| إصابة مخبأ | بلا نداء ناشر |

## تمارين

### تمرين سهل

لماذا `await expect(promise).rejects`؟

<details>
<summary>الحل</summary>

لأن نسيان الانتظار لا يفشل الاختبار عندما يُرفض الوعد بعد انتهاء الدالة.

</details>

### تمرين متوسط

ماذا تثبت حالة المخبأ التي تعيد `{ title }`؟

<details>
<summary>الحل</summary>

أن `lookup` لا يستدعي `set` ولا الناشر عند الإصابة.

</details>

### تمرين متقدم

أي حقول الحمولة تكفي لتوقع `add`، وما الذي يجب ألا يوجد؟

<details>
<summary>الحل</summary>

`bookId` و `coverKey` وربما `requestId`. لا كائن `book` من Prisma ولا `Buffer`.

</details>

## أسئلة مراجعة

- كم فرعًا في حالة واحدة؟
- كيف تُكتب الأوهام في قالب ESM؟
- أي عنوان قاعدة لا يظهر في هذا الملف؟

## الخلاصة

اختبار الخدمة حالات صغيرة على أوهام: أثر الحفظ والإعلان والمخبأ والطابور. انتظر توقع الرفض، ولا تعتمد على قاعدة المطور.

## ماذا بعد؟

المتحكم مع خدمة مستبدَلة: هل يمرر المعرف والجسم كما يجب.

## المصادر الرسمية

- NestJS — Testing: https://docs.nestjs.com/fundamentals/testing
