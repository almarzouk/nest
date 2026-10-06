# Media Type Versioning

## الهدف من هذا الدرس

اختيار عقد كتاب رف من ترويسة `Accept`، بالنمط الذي توثقه Nest: نوع الوسائط ثم فاصلة منقوطة ثم مفتاح النسخة، مثل `application/json;v=2`.

## المشكلة التي يحلها

الشريك يعتبر النسخة جزءًا من شكل التمثيل لا جزءًا من اسم المورد ولا ترويسة خاصة برف. يرسل `Accept` أصلًا لطلب JSON. تريد أن تحمل نفس الترويسة رقم العقد، وأن يفشل الطلب غير المفهومة نسخته بـ 404.

## الفكرة ببساطة

`VersioningType.MEDIA_TYPE` يقرأ النسخة من `Accept`. الخيار `key` هو المفتاح مع الفاصل. للمثال الرسمي `Accept: application/json;v=2` تكون `key: 'v='`. المتحكم ما زال يعلن `version: '2'`.

## التعريف التقني

```ts
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
```

الصفحة تقول إن النسخة تُفصل عن نوع الوسائط بفاصلة منقوطة، والمفتاح يُعامل كبادئة للنسخة ولذلك يجب أن يتضمن المفتاح والفاصل. `v=` تعني أن القيمة بعد علامة يساوي هي النسخة. مسار بلا نسخة في `Accept`، أو `Accept` غائب، يعيد 404 إن لم تضع `defaultVersion`. هذا ما تغطيه اختبارات Media Type في الإطار: `application/json;v=1` يطابق، و `application/json` بلا معلمة يعيد 404 من غير افتراضي. `@Version` والحياد و `defaultVersion` يعملون كما في بقية الأنواع. الوسيط يستطيع تقييد `version` أيضًا.

## Mental Model

```text
Accept: application/json;v=2
key = "v="
النسخة المستخرجة = 2
→ متحكم version: '2'
```

## أبسط مثال

```ts
import { Controller, Get } from '@nestjs/common';

@Controller({ path: 'books', version: '2' })
export class BooksMediaV2Controller {
  @Get()
  list() {
    return [{ id: 'b1', price: { amount: 2500, currency: 'SAR' } }];
  }
}
```

```http
GET /books HTTP/1.1
Accept: application/json;v=2
```

## شرح المثال

- `key: 'v='` ليس رقم النسخة. الرقم في الترويسة وفي `version` على المتحكم.
- نسيان الفاصلة في `key` (`v` بدل `v=`) لا يطابق المثال الموثق.
- الجسم ما زال JSON. الذي تغيّر هو معلمة النوع التي يختار بها Nest الدالة.

## ماذا يحدث داخليًا؟

1. المحول يمرر `Accept` إلى مستخرج Media Type.
2. Nest يبحث عن المفتاح `v=` ويأخذ ما بعده كنسخة.
3. يطابقها مع metadata المسار.
4. إن وُجدت نسختان في التصميم، فكل واحدة متحكم أو `@Version` على دالتين تشتركان في المسار.
5. عدم وجود المعلمة يعيد 404 بلا `defaultVersion`، فلا تسقط النسخة الجديدة فوق عميل طلب `application/json` فقط.

## مثال عملي

الإبقاء على الجوال الذي يرسل `Accept: application/json` بلا معلمة خلال الانتقال:

```ts
import { VersioningType } from '@nestjs/common';

app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
  defaultVersion: '1',
});
```

بعد تحديث الشريك، احذف الافتراضي. اختبار القبول: `Accept: application/json;v=2` يعيد كائن السعر، و `Accept: application/json;v=9` يعيد 404، و `Accept: application/json` يعيد 404 بعد حذف الافتراضي.

## أخطاء شائعة

- وضع النسخة في `Content-Type` الخاص بالطلب وانتظار أن يقرأها هذا النوع. التوثيق يتحدث عن `Accept`.
- `key: 'v'` بلا `=` بينما الأمثلة كلها `v=`.
- كاش أو متصفح يعيد استجابة v1 لطلب `Accept` مختلف لأن `Vary: Accept` غائب.
- اعتبار Media Type بديلًا عن `Content-Type: application/json` على الجسم. ما زلت تعلن نوع الجسم الذي ترسله.

## متى نستخدمه؟

عندما يكون عقد التمثيل جزءًا من تفاوض المحتوى مع شريك معتاد على `Accept`، والمسار يجب أن يبقى `/books`.

## متى لا نستخدمه؟

لعملاء لا يضبطون `Accept` بدقة (بعض أدوات التجربة تضع `*/*` فقط). URI أظهر لهم. ولا تخلط الأنواع الثلاثة في تطبيق واحد من غير نوع Custom موثّق.

## مقارنة

| الطلب `Accept` | بلا `defaultVersion` | مع `defaultVersion: '1'` |
| --- | --- | --- |
| `application/json;v=2` | v2 إن وُجدت | v2 |
| `application/json;v=9` | 404 | 404 |
| `application/json` | 404 | v1 |
| بلا ترويسة | 404 | v1 |

## تمارين

### تمرين سهل

ما قيمة `key` لمثال التوثيق `application/json;v=2`؟

<details>
<summary>الحل</summary>

`'v='`. المفتاح يشمل علامة يساوي لأن الصفحة تعامل المفتاح كبادئة تفصل النسخة.

</details>

### تمرين متوسط

شريك يصر أن `Accept: application/json` يجب أن يبقى على العقد القديم شهرًا، ثم يصبح خطأً. ما الخيار الذي تضيفه ثم تحذفه؟

<details>
<summary>الحل</summary>

`defaultVersion: '1'` خلال الشهر. بعده احذفه ليصبح الطلب بلا `v=` استجابة 404 كما في اختبارات الصفحة عند غياب النسخة.

</details>

### تمرين متقدم

لماذا `Vary` جزء من هذا النوع أكثر مما هو في URI؟

<details>
<summary>الحل</summary>

لأن URI يغيّر العنوان فمفتاح الكاش يختلف وحده. هنا العنوان واحد ويختلف `Accept`. من غير `Vary: Accept` قد تُخزَّن نسخة عقد وتُعاد لعقد آخر على نفس المسار.

</details>

## أسئلة مراجعة

- أي ترويسة يقرأ هذا النوع؟
- ماذا يتضمن `key` غير اسم الحرف؟
- ما وضع `application/json` بلا معلمة إن لم يوجد افتراضي؟
- هل `@Version` يعمل هنا كما في URI؟

## الخلاصة

Media Type Versioning يستخرج النسخة من `Accept` بمفتاح مثل `v=`. المسار ثابت، والعقد في معلمة النوع. الغياب وعدم التطابق 404 ما لم تضع `defaultVersion`.

## ماذا بعد؟

الآلية جاهزة. فصل تصميم الـ API يقرر الإصدار كجزء من العقد المكتوب للعميل، لا كخيار إطار فقط.

## المصادر الرسمية

- NestJS — Versioning: https://docs.nestjs.com/techniques/versioning
- MDN — Accept: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Accept
