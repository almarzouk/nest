# Header Versioning

## الهدف من هذا الدرس

خدمة نسختي كتاب رف على نفس المسار `/books`، والنسخة تأتي في ترويسة تسميها أنت عند `enableVersioning`.

## المشكلة التي يحلها

روابط الكتالوج مشتركة في رسائل المتجر: `https://api.raf.example/books/b1`. تغييرها إلى `/v2/books/b1` يكسر الإشارات المحفوظة. العميل يستطيع إرسال ترويسة، والمسار يبقى.

## الفكرة ببساطة

`VersioningType.HEADER` يقرأ النسخة من اسم ترويسة تحدده في `header`. المسار في الديكور لا يحتوي رقمًا. `@Controller({ version: '1' })` ما زال يعلن أي الدوال تخدم تلك القيمة. طلب بلا الترويسة 404 إن لم يوجد `defaultVersion`.

## التعريف التقني

```ts
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'X-Api-Version',
});
```

`header` اسم الترويسة كما في الصفحة. القيمة `'1'` أو `'2'` تطابق `version` في المتحكم أو `@Version`. لا تُضاف بادئة `v` لأن البادئة سلوك URI. وسائط Nest تستطيع استهداف نسخة: `forRoutes({ path: 'books', method: RequestMethod.GET, version: '2' })`. التخزين المؤقت أمام المسار يجب أن يغيّر مفتاحه حسب الترويسة، وإلا تُخدم v2 من نسخة v1 المخزنة.

## Mental Model

```text
GET /books/b1
X-Api-Version: 2
→ BooksV2Controller
```

نفس السطر بلا الترويسة لا يختار v2 «بالتقدير».

## أبسط مثال

```ts
import { Controller, Get, Param } from '@nestjs/common';
import { BooksService } from './books.service.js';
import { toV2 } from './book.presenters.js';

@Controller({ path: 'books', version: '2' })
export class BooksHeaderV2Controller {
  constructor(private readonly books: BooksService) {}

  @Get(':id')
  async find(@Param('id') id: string) {
    return toV2(await this.books.find(id));
  }
}
```

الطلب: `X-Api-Version: 2`. العنوان يبقى `/books/:id`.

## شرح المثال

- نوع الإصدار عام للتطبيق. لا تخلط URI و Header في نفس `enableVersioning` إلا عبر النوع Custom، وهو خارج هذا الدرس.
- متحكم `version: '1'` على `path: 'books'` يخدم نفس العنوان عندما تكون الترويسة `1`.
- العميل الذي لا يرسل الترويسة يحتاج `defaultVersion` إن أردت إبقاء السلوك القديم أثناء الانتقال.

## ماذا يحدث داخليًا؟

1. Nest يقرأ الترويسة المسماة في `header`.
2. يطابق القيمة مع metadata النسخة على المتحكم أو الدالة.
3. `@Version` على الدالة يتجاوز نسخة الصنف، كما في كل الأنواع.
4. عدم التطابق أو غياب النسخة بلا افتراضي يعيد 404، والنص في الصفحة واحد لكل الأنواع.
5. الوسيط الذي قيّد `version: '2'` لا يعمل على طلب ترويسته `1`.

## مثال عملي

إبقاء الجوال القديم بلا تعديل ترويسة خلال شهر، عبر الافتراضي:

```ts
import { VersioningType } from '@nestjs/common';

app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'X-Api-Version',
  defaultVersion: '1',
});
```

الجوال الحالي ينادي `/books/b1` بلا ترويسة فيحصل على `toV1`. الموقع يرسل `X-Api-Version: 2`. عندما يُحدَّث الجوال، أزل `defaultVersion` حتى يظهر من نسي الترويسة كـ 404 في الاختبارات لا كعقد قديم صامت.

## أخطاء شائعة

- تسمية الترويسة `version` في الكود و `X-Api-Version` في الدليل، أو العكس.
- افتراض أن المتصفح يرسل الترويسة في انتقال عادي. صفحة تفتح الرابط لا تضيف ترويسة مخصصة؛ هذا النمط لعميل API.
- CDN يخزن `/books/b1` من غير تضمين الترويسة في مفتاح التخزين.
- وضع `v2` في الترويسة بينما `version` في المتحكم `'2'`.

## متى نستخدمه؟

عملاء مبرمجون يقدرون على ترويسة، وعناوين تريد إبقاءها ثابتة بين العقود.

## متى لا نستخدمه؟

لروابط تُفتح في المتصفح مباشرة، أو لشريك لا يتحكم في الترويسات. عندها URI أوضح.

## مقارنة

| | URI | Header |
| --- | --- | --- |
| أين النسخة | المسار | ترويسة تحدد اسمها |
| العنوان بين v1 و v2 | مختلف | نفسه |
| طلب بلا نسخة | 404 أو افتراضي أو neutral | 404 أو `defaultVersion` |
| بادئة `v` | نعم افتراضيًا | لا |

## تمارين

### تمرين سهل

`header: 'X-Api-Version'` والمتحكم `version: '2'`. أي طلب يصل إلى `toV2`؟

<details>
<summary>الحل</summary>

`GET /books/...` مع `X-Api-Version: 2`. المسار لا يحتوي `v2`.

</details>

### تمرين متوسط

لماذا قد يرى عميل v2 جسم v1 من وراء الكاش؟

<details>
<summary>الحل</summary>

لأن المفتاح كان المسار فقط. يجب أن يدخل `X-Api-Version` في مفتاح التخزين أو في `Vary` حتى لا تُعاد نسخة عقد آخر.

</details>

### تمرين متقدم

بعد شهر الانتقال تريد اكتشاف الجوالات التي لم تُحدَّث. ماذا تغيّر في `enableVersioning`؟

<details>
<summary>الحل</summary>

أزل `defaultVersion`. الطلب بلا `X-Api-Version` يصبح 404 حسب الصفحة، فيظهر في السجل كعميل لم ينتقل، بدل أن يبقى على v1 بصمت.

</details>

## أسئلة مراجعة

- أي حقل في الخيارات يسمي الترويسة؟
- هل حرف `v` يُضاف إلى قيمة الترويسة؟
- ماذا يفعل `defaultVersion` هنا؟
- لماذا المتصفح العادي ليس العميل النموذجي لهذا النوع؟

## الخلاصة

Header Versioning يثبت المسار وينقل رقم العقد إلى ترويسة تسميها `header`. القيمة تطابق `version` في الديكور. الغياب 404 بلا `defaultVersion`، والكاش يجب أن يفرّق الترويسة.

## ماذا بعد؟

عميل يريد النسخة داخل تفاوض المحتوى في `Accept` لا في ترويسة مخصصة. الدرس التالي Media Type.

## المصادر الرسمية

- NestJS — Versioning: https://docs.nestjs.com/techniques/versioning
