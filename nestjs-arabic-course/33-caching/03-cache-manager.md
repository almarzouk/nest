# cache-manager

## الهدف من هذا الدرس

قراءة المفتاح وكتابته عبر مدير المخبأ المحقون حول نداء ISBN الناجح.

## المشكلة التي يحلها

الوحدة مسجّلة والناشر ما زال يُستدعى كل مرة. من غير `get` قبل الحساب و `set` بعد النجاح لا يوجد مخبأ، فقط مزوّد فارغ.

## الفكرة ببساطة

تحقن `CACHE_MANAGER` ونوع المدير من `cache-manager`. المفتاح سلسلة ثابتة الشكل. `get` يعيد القيمة أو `undefined` عند الغياب. `set` يكتب بعد أن تحصل على `{ title }` أو `null` المتعمد. الاستثناء يخرج قبل `set`.

## التعريف التقني

```ts
import { Inject, Injectable } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class IsbnService {
  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}

  private key(isbn: string) {
    return `isbn:${isbn}`;
  }
}
```

العمليات التي تهم الآن:

```ts
const hit = await this.cache.get<{ title: string } | null>(this.key(isbn));
await this.cache.set(this.key(isbn), { title }, 10 * 60 * 1000);
```

الوسيط الثالث لـ `set` مدة هذا المفتاح بالميلي ثانية وتغلّب الافتراض. `del` يحذف مفتاحًا إن عرفت أن الناشر حدّث العنوان فورًا.

البادئة `isbn:` تمنع الاصطدام مع مفتاح `book:` لاحقًا على المخزن نفسه.

## Mental Model

```text
get(isbn:...)
  موجود -> أعده
  غير موجود -> lookup الشبكي
    نجاح أو null متعمد -> set
    رمي -> بلا set
```

## أبسط مثال

```ts
async lookup(isbn: string): Promise<{ title: string } | null> {
  const key = this.key(isbn);
  const cached = await this.cache.get<{ title: string } | null>(key);
  if (cached !== undefined) {
    return cached;
  }
  const fresh = await this.fetchFromPublisher(isbn);
  await this.cache.set(key, fresh, 10 * 60 * 1000);
  return fresh;
}
```

`fetchFromPublisher` هي الدالة التي ترمي عند المهلة وتعيد `null` عند `404` كما في الفصل السابق. إن رمت، لا يصل التنفيذ إلى `set`.

## شرح المثال

`undefined` من `get` يعني إخفاقًا. `null` المخزّن يعني إصابة بأن الناشر لا يعرف الرقم. الشرط `cached !== undefined` يفرّق الحالين. لو كتبت `if (cached)` لأعاد الطلب عند `null` وفي كل مرة يُضرب الناشر بالرقم الخاطئ، أو لتجاهلت عنوانًا فارغًا بشكل خاطئ.

## ماذا يحدث داخليًا؟

`cache-manager` يمرر الأمر إلى المخزن، والذاكرة في هذا الدرس. `set` مع TTL يسجل انتهاء المفتاح. `get` بعد الانتهاء يتصرف كغياب. العمليات `async` حتى عندما يكون المخزن ذاكرة، لأن الواجهة نفسها تخدم Redis لاحقًا. لا تبنِ المفتاح من كائن طلب كامل. `isbn` نص متحقق قبل ذلك حتى لا تصبح مسافة أو ترميز مختلف مفتاحين للرقم نفسه.

## مثال عملي

بعد أن يصحح الناشر عنوانًا وتريد تجاوز المدة:

```ts
await this.cache.del(`isbn:${isbn}`);
```

الاستدعاء من مسار إداري بعد تحديث تعرفه، لا من كل `GET` عام. الحذف التالي يعيد `fetchFromPublisher` مرة واحدة.

## أخطاء شائعة

- `if (!cached)` فيضيع تخزين `null`.
- `set` داخل `catch`.
- مفتاح `isbn` بلا بادئة.
- حقن المدير في المتحكم وتخزين جسم HTTP فيه جلسة.

## متى نستخدمه؟

حول حساب واضح المفتاح: `lookup` ولاحقًا قائمة منشورة ثقيلة تبنيها خدمة.

## متى لا نستخدمه؟

كغلاف على `BooksService.create`. الكتابة تمر على القاعدة ثم تحذف المفتاح المتأثر إن كانت القراءة مخزّنة. التخزين لا يستبدل `insert`.

## مقارنة

| قيمة `get` | المعنى |
| --- | --- |
| `undefined` | لا نسخة، احسب |
| `null` | نسخة تقول إن الناشر أعاد غيابًا |
| `{ title }` | نسخة العنوان |

## تمارين

### تمرين سهل

أي رمز تحقنه للحصول على المدير؟

<details>
<summary>الحل</summary>

`CACHE_MANAGER` من `@nestjs/cache-manager`.

</details>

### تمرين متوسط

لماذا `null` المخزّن ليس إخفاقًا؟

<details>
<summary>الحل</summary>

لأنه نتيجة `404` متعمدة. `undefined` وحده هو الغياب من المخزن.

</details>

### تمرين متقدم

أين يُستدعى `del` بالنسبة لتحديث عنوان تعرفه من لوحة الناشر؟

<details>
<summary>الحل</summary>

بعد أن تتأكد من التحديث، على المفتاح `isbn:${isbn}`، حتى القراءة التالية تحسب من جديد ولا تنتظر TTL.

</details>

## أسئلة مراجعة

- ما وحدة المدة في `set`؟
- لماذا البادئة في المفتاح؟
- ماذا لا يُكتب في فرع الخطأ؟

## الخلاصة

`get` ثم حساب ثم `set` للنجاح و `null` المتعمد. ميّز `undefined` عن `null`. احذف المفتاح عندما تعرف أن المصدر تغيّر الآن.

## ماذا بعد؟

كيف تختار الرقم الذي تمرره مدة حياة، وماذا يحدث عند انتهائها.

## المصادر الرسمية

- NestJS — Caching: https://docs.nestjs.com/data/caching
- cache-manager: https://github.com/jaredwray/cache-manager
