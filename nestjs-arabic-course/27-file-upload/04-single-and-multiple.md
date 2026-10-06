# ملف واحد وملفات متعددة

## الهدف من هذا الدرس

اختيار المعترض حسب شكل النموذج: ملف واحد، مصفوفة باسم واحد، حقول مسماة، أي حقل، أو نموذج بلا ملفات.

## المشكلة التي يحلها

`FileInterceptor` يمسك حقلًا واحدًا وملفًا واحدًا. نموذج «صور المعرض» يكرر الاسم `photos`. نموذج الكتاب يرسل `cover` و `sample` معًا. معترض الحقل الواحد يترك الباقي خارج العقد أو يرفضه بشكل يصعب شرحه للعميل.

## الفكرة ببساطة

الاسم في الزينة يصف شكل الأجزاء، لا نوع الملف. بعد الاختيار، `@UploadedFile` للواحد و `@UploadedFiles` للباقي. كل معترض يقبل خيارات Multer نفسها: قرص، وحد حجم، حتى لا ترجع المصفوفة الكبيرة إلى `buffer`.

## التعريف التقني

من `@nestjs/platform-express`، كما في صفحة الرفع:

| الزينة | شكل النموذج | الاستخراج |
| --- | --- | --- |
| `FileInterceptor(field)` | ملف واحد في حقل | `@UploadedFile()` |
| `FilesInterceptor(field, maxCount?, options?)` | عدة ملفات بالاسم نفسه | `@UploadedFiles()` مصفوفة |
| `FileFieldsInterceptor([{ name, maxCount? }], options?)` | حقول مختلفة | `@UploadedFiles()` كائن مفاتيحه أسماء الحقول |
| `AnyFilesInterceptor(options?)` | أي أسماء حقول | `@UploadedFiles()` مصفوفة، وكل عنصر فيه `fieldname` |
| `NoFilesInterceptor(options?)` | نص multipart فقط | `@Body()`، وأي ملف يرفض بـ `BadRequestException` |

`maxCount` سقف الحقل. `limits.files` سقف الطلب كله من جهة Multer.

## Mental Model

```text
غلاف واحد          FileInterceptor('cover')
معرض 1..6          FilesInterceptor('photos', 6)
غلاف + عينة PDF    FileFieldsInterceptor([{ name:'cover', maxCount:1 }, { name:'sample', maxCount:1 }])
صندوق بلا عقد      AnyFilesInterceptor  — نادر، لأن الأسماء تصبح بيانات عميل
نموذج بلا ملف      NoFilesInterceptor
```

## أبسط مثال

غلاف وصورة خلفية، كل منهما ملف واحد على القرص:

```ts
import { Controller, Post, UploadedFiles, UseInterceptors } from '@nestjs/common';
import { FileFieldsInterceptor } from '@nestjs/platform-express';
import { coverLimits } from './cover-limits.js';

@Controller('books')
export class GalleryController {
  @Post(':id/images')
  @UseInterceptors(FileFieldsInterceptor([
    { name: 'cover', maxCount: 1 },
    { name: 'sample', maxCount: 1 },
  ], coverLimits))
  save(@UploadedFiles() files: {
    cover?: Express.Multer.File[];
    sample?: Express.Multer.File[];
  }) {
    return {
      coverKey: files.cover?.[0]?.filename ?? null,
      sampleKey: files.sample?.[0]?.filename ?? null,
    };
  }
}
```

`FileFieldsInterceptor` يعيد مصفوفة حتى لو كان `maxCount` واحدًا. أول عنصر هو الملف.

## شرح المثال

الخيارات المشتركة `coverLimits` تطبَّق على الحقلين: القرص وحد الحجم. المفتاح `cover` قد يغيب إن لم يرسل العميل الجزء، لذلك `?.` و `null` في العقد أوضح من رمي استثناء داخل التعبير. إن أرسل العميل ملفين في `cover`، `maxCount: 1` يوقف الزيادة.

معرض بالاسم نفسه يبدّل الزينة فقط:

```ts
@UseInterceptors(FilesInterceptor('photos', 6, coverLimits))
add(@UploadedFiles() photos: Express.Multer.File[]) {
  return { keys: photos.map((photo) => photo.filename) };
}
```

## ماذا يحدث داخليًا؟

كل زينة تبني وسيط Multer بشكل مختلف. `FilesInterceptor` يجمع الأجزاء ذات `fieldname` الواحد في قائمة ويقطع عند `maxCount`. `FileFieldsInterceptor` يوزع الأجزاء على خريطة بالاسم، والحقول غير المذكورة لا تدخل الخريطة. `AnyFilesInterceptor` يجمع كل جزء ملف في قائمة واحدة، فيصبح اسم الحقل جزءًا من بيانات العميل يجب ألا يُستخدم مسارًا. `NoFilesInterceptor` يملأ `@Body()` من الحقول النصية ويرفض إن ظهر جزء فيه `filename`.

التحليل ما زال يكتمل قبل الدالة على Express. ستة ملفات في الذاكرة تعني ستة `buffer`. مرّر `storage` هنا كما في الملف الواحد.

## مثال عملي

عقد إنشاء كتاب: عنوان نصي، وغلاف إجباري، وعينة اختيارية.

```text
title   نص     @Body('title')
cover   1 ملف  files.cover[0]
sample  0..1   files.sample?.[0]
```

الخدمة ترمي `BadRequestException` إن غاب `cover`. `AnyFilesInterceptor` يضعف العقد لأن العميل يسمّي حقلًا ثالثًا فيُخزَّن مع الكتاب.

## أخطاء شائعة

- استخدام `@UploadedFile` مع `FilesInterceptor`. الناتج مصفوفة، والزينة المفردة لا تطابقها.
- نسيان أن `FileFieldsInterceptor` يعيد `File[]` لا `File`.
- `AnyFilesInterceptor` ثم `join(uploadDir, file.fieldname)`.
- رفع السقف إلى عشرات الملفات مع تخزين في الذاكرة.

## متى نستخدمه؟

`FileFieldsInterceptor` حين تكون الأسماء جزءًا من تصميمك. `FilesInterceptor` لمعرض. `NoFilesInterceptor` لنموذج غني بلا مرفقات حتى لا يمر ملف صامت إلى `@Body()`.

## متى لا نستخدمه؟

`AnyFilesInterceptor` كاختصار دائم. الأسماء الحرة تصعّب الحد والاختبار. حوّلها إلى حقول معلنة عندما يستقر المنتج.

## مقارنة

| الحاجة | الزينة | شكل القيمة |
| --- | --- | --- |
| غلاف فقط | `FileInterceptor` | ملف أو `undefined` |
| حتى 6 صور | `FilesInterceptor('photos', 6)` | `File[]` |
| غلاف وعينة | `FileFieldsInterceptor` | `{ cover?: File[]; sample?: File[] }` |

## تمارين

### تمرين سهل

أي زينة لمعرض حقوله كلها باسم `photos`؟

<details>
<summary>الحل</summary>
`FilesInterceptor('photos', maxCount, options)` مع `@UploadedFiles()`.
</details>

### تمرين متوسط

لماذا `files.cover` مصفوفة رغم `maxCount: 1`؟

<details>
<summary>الحل</summary>
لأن `FileFieldsInterceptor` يجمّع كل حقل في قائمة. السقف يحد الطول، ولا يغيّر النوع إلى ملف مفرد.
</details>

### تمرين متقدم

متى تختار `NoFilesInterceptor` بدل ترك المسار على `FileInterceptor`؟

<details>
<summary>الحل</summary>
عندما يكون العقد حقولًا نصية بصيغة multipart، وتريد رفض أي جزء ملف بـ `400` بدل تخزينه صامتًا.
</details>

## أسئلة مراجعة

- ما فرق `maxCount` عن `limits.files`؟
- أي زينة ترفض وجود ملف؟
- لماذا الحقول المعلنة أوضح من `AnyFilesInterceptor`؟

## الخلاصة

اختر المعترض من شكل النموذج. الواحد مع `@UploadedFile`، والبقية مع `@UploadedFiles`، والقرص والحد يبقيان على الخيارات مهما كثر العدد.

## ماذا بعد؟

الحجم و MIME: رفض المحتوى بعد أن صار شكل الحقول معروفًا، ومن غير الاكتفاء بالامتداد.

## المصادر الرسمية

- NestJS — File upload: https://docs.nestjs.com/http/file-upload
