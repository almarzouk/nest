# Multipart

## الهدف من هذا الدرس

قراءة طلب `multipart/form-data`: أجزاء نصية وملف ثنائي في رسالة HTTP واحدة، قبل أي حديث عن Multer أو المسار على القرص.

## المشكلة التي يحلها

غلاف الكتاب ثنائي. وضعه داخل JSON يجبرك على `base64`، فيكبر الجسم نحو الثلث، ويمر على `ValidationPipe` كأنه نص. جسم `application/octet-stream` يحمل ملفًا واحدًا بلا حقول مرافقة مثل `bookId`. النموذج الذي يرسل العنوان والصورة معًا يستخدم نوعًا آخر: `multipart/form-data`.

## الفكرة ببساطة

الطلب يُقسَم بسلسلة فاصلة اسمها `boundary`. كل جزء إما حقل نص وإما ملف. الجزء يعلن اسم الحقل في `Content-Disposition`. الملف يضيف `filename` ونوع محتوى خاصًا به. الخادم لا يرى «كائنًا». يرى تيارًا يجب أن يُحلَّل قبل أن يصل المعالج.

## التعريف التقني

ترويسة الطلب تشبه:

```text
Content-Type: multipart/form-data; boundary=----BookBoundary
```

ثم الجسم، مختصرًا:

```text
------BookBoundary
Content-Disposition: form-data; name="title"

NestJS in Practice
------BookBoundary
Content-Disposition: form-data; name="cover"; filename="cover.png"
Content-Type: image/png

<bytes>
------BookBoundary--
```

- `name` هو مفتاح الحقل. في المثال: `title` و `cover`.
- `filename` يميّز جزء الملف عن جزء النص.
- `Content-Type` داخل الجزء يعلنه العميل. هو ادّعاء، لا فحصًا للمحتوى.
- الحد الفاصل الأخير ينتهي بـ `--`.

`express.json()` و `ValidationPipe` على DTO لا يفكّان هذا الشكل. Multer، في صفحة الرفع، يتعامل مع `multipart/form-data` فقط.

## Mental Model

```text
عميل (نموذج)
  boundary
    جزء name="title"          نص
    جزء name="cover"          ملف + filename
  محلل multipart              يفصل الأجزاء
  المعالج                     يرى الحقول والملف جاهزين
```

JSON حقول. Multipart حقول وملفات. التحليل خطوة قبل الدالة.

## أبسط مثال

نموذج يطابق الحقل الذي ستربطه لاحقًا بـ `FileInterceptor('cover')`.

```html
<form action="/books/42/cover" method="post" enctype="multipart/form-data">
  <input name="title" value="NestJS in Practice" />
  <input name="cover" type="file" accept="image/png" />
  <button type="submit">رفع</button>
</form>
```

الأمر نفسه من الطرفية:

```bash
curl -F "title=NestJS in Practice" \
  -F "cover=@./cover.png;type=image/png" \
  http://localhost:3000/books/42/cover
```

`-F` في curl يضبط `multipart/form-data` ويولّد الـ boundary. `-d` يرسل جسمًا آخر، فالمحلل الخاص بالملفات يرفضه.

## شرح المثال

`enctype` هو العقد. بدونه يرسل المتصفح `application/x-www-form-urlencoded`، فيضيع الملف أو يتحول إلى اسم فقط. اسم `cover` سيصبح `fieldName` في المعترض. اسم `title` سيظهر لاحقًا في `@Body()` بعد التحليل. الامتداد في `accept` توجيه للمتصفح، والخادم ما زال يرى بايتات.

## ماذا يحدث داخليًا؟

العميل يكتب الأجزاء متتابعة على الاتصال. المحلل يبحث عن الـ boundary، يقرأ ترويسات الجزء، ثم ينسخ البايتات حتى الفاصل التالي. الحقول النصية تُجمع في جسم. جزء الملف يُمسك حتى يكتمل أو يُرفض بسبب حد. إن غاب المحلل، يصل الجسم خامًا إلى مسار يتوقع JSON، فيفشل التحليل أو يُتجاهل الملف.

ترتيب الأجزاء قرار للعميل. في رفع Express المعتاد يكتمل التحليل قبل المعالج. التدفق أثناء الوصول، على Fastify، هو `FileStreamInterceptor`.

## مثال عملي

عقد غلاف: `title` و `notes` نص يقرؤهما `@Body()`، و `cover` ملف واحد يقرؤه `@UploadedFile()`. المعالج لا يقصّ الـ boundary بيده.

## أخطاء شائعة

- إرسال الملف مع `Content-Type: application/json`.
- تسمية الحقل `file` في النموذج وانتظار `cover` في المعترض.
- الوثوق بـ `Content-Type` داخل الجزء كأنه فحص سحري للمحتوى.
- وضع صورة كبيرة كـ `base64` داخل JSON لأن «أبسط في Postman».

## متى نستخدمه؟

عندما يجتمع نص وملف في عملية واحدة: غلاف، مرفق تذكرة، شعار منظمة.

## متى لا نستخدمه؟

لقائمة كتب بلا ملف. JSON يبقى العقد. ولرفع مباشر من المتصفح إلى مخزن كائن موقّع، الرسالة لا تمر أصلًا على Nest.

## مقارنة

| الشكل | ماذا يحمل | من يحلله في Nest المعتاد |
| --- | --- | --- |
| `application/json` | حقول | `express.json` ثم `ValidationPipe` |
| `multipart/form-data` | حقول وملفات | Multer عبر معترض الملف |
| `application/octet-stream` | بايتات بلا أسماء حقول | جسم خام، عقد مختلف |

## تمارين

### تمرين سهل

ما الذي يفصل أجزاء الجسم بعضها عن بعض؟

<details>
<summary>الحل</summary>

قيمة `boundary` المعلنة في `Content-Type`.

</details>

### تمرين متوسط

لماذا `-d '{"cover":"..."}'` لا يغني عن `-F`؟

<details>
<summary>الحل</summary>

لأن `-d` لا يبني `multipart/form-data`. المحلل الذي تنتظره معترضات الملفات يشترط هذا النوع.

</details>

### تمرين متقدم

جزء يحمل `filename` و `Content-Type: image/png` واسمه `cover`. ماذا يثبت ذلك عن البايتات؟

<details>
<summary>الحل</summary>

يثبت ادّعاء العميل فقط: اسم الحقل، واسم الملف، والنوع المعلن. البايتات قد تكون غير صورة. الفحص لاحق.

</details>

## أسئلة مراجعة

- ما الفرق بين جزء نصي وجزء ملف في `Content-Disposition`؟
- لماذا لا يكفي `ValidationPipe` هنا؟
- أي حقل في النموذج يجب أن يطابق `fieldName` لاحقًا؟

## الخلاصة

Multipart رسالة متعددة الأجزاء بفاصل معلن. الحقول النصية والملف يسافران معًا، والتحليل شرط قبل أي تخزين.

## ماذا بعد؟

Multer: المحلل الذي يقرأ هذه الرسالة على Express، وأين تذهب البايتات إن لم تختر قرصًا.

## المصادر الرسمية

- NestJS — File upload: https://docs.nestjs.com/http/file-upload
- MDN — `multipart/form-data`: https://developer.mozilla.org/docs/Web/HTTP/Reference/Methods/POST
