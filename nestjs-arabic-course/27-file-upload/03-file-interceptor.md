# FileInterceptor

## الهدف من هذا الدرس

ربط حقل ملف واحد بمسار: `FileInterceptor` يركّب Multer، و `@UploadedFile` يسلّم الكائن إلى المعالج.

## المشكلة التي يحلها

معرفة Multer لا تضع الوسيط على `POST /books/:id/cover` وحدها. من غير معترض، جسم multipart يظل خارج معاملات Nest. المعالج إما لا يرى الملف، وإما يقرأ `req` الخام ويتجاوز الأنابيب.

## الفكرة ببساطة

`FileInterceptor('cover')` اسم الحقل أول وسيط، وخيارات Multer وسيطًا ثانيًا اختياريًا. الزينة `@UseInterceptors` تعلّقه على الدالة أو على المتحكم. `@UploadedFile()` من `@nestjs/common` يستخرج الملف بعد أن ينجح التحليل.

## التعريف التقني

`FileInterceptor` مصدَّر من `@nestjs/platform-express`. التوقيع العملي: `FileInterceptor(fieldName, options?)`. `options` من نوع `MulterOptions`، وهو كائن منشئ Multer نفسه: `storage` و `limits` و `fileFilter`.

الكائن على Express من نوع `Express.Multer.File` بعد تثبيت `@types/multer`. حقول تكثر رؤيتها:

| الحقل | معناه |
| --- | --- |
| `fieldname` | اسم الحقل، مثل `cover` |
| `originalname` | الاسم كما أرسله العميل |
| `mimetype` | النوع المعلن |
| `size` | الحجم بالبايت |
| `buffer` | موجود مع التخزين في الذاكرة |
| `path` / `filename` | موجودان مع تخزين القرص |

المعترض يعمل بعد اختيار الراوت وقبل جسم الدالة. فشل Multer لا يستدعي الدالة.

## Mental Model

```text
POST /books/42/cover
  FileInterceptor('cover', coverLimits)
    يقرأ جزء name="cover"
    يطبّق storage و limits
  @UploadedFile()  ->  Express.Multer.File | undefined
  CoverService يحفظ المسار بجانب الكتاب 42
```

الحقول النصية في النموذج نفسه تبقى عمل `@Body()`.

## أبسط مثال

```ts
import { Controller, Param, Post, UploadedFile, UseInterceptors } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { coverLimits } from './cover-limits.js';
import { CoversService } from './covers.service.js';

@Controller('books')
export class CoversController {
  constructor(private readonly covers: CoversService) {}

  @Post(':id/cover')
  @UseInterceptors(FileInterceptor('cover', coverLimits))
  replace(
    @Param('id') bookId: string,
    @UploadedFile() file?: Express.Multer.File,
  ) {
    return this.covers.replace(bookId, file);
  }
}
```

`coverLimits` هو كائن الدرس السابق: `diskStorage` و `fileSize`.

## شرح المثال

الوسيطة الأولى `'cover'` يجب أن تطابق `name` في النموذج. الوسيطة الثانية تغلّب افتراض `MulterModule` لهذا المسار فقط. المعالج لا يستورد `multer`. هو يستلم ناتج التحليل. `@Param('id')` مستقل عن الملف: معرف الكتاب في المسار، والبايتات في الجزء.

إن غاب الجزء، تكون قيمة المعامل `undefined` ما لم يرفض `fileFilter` أو أنبوب لاحق الطلب. لا تنادِ `file.path` من غير هذا الفحص.

## ماذا يحدث داخليًا؟

Nest، عند الإقلاع، ترى `@UseInterceptors` وتبني وسيط Multer لحقل واحد. طلب بغير `multipart/form-data` لا يمر من هذا المحلل كما تتوقع صفحة الرفع. طلب بالحقل الصحيح يُملأ ملفه حسب `storage`. ثم يُستدعى المعالج ومعامل الزينة يشير إلى ذلك الكائن. إرجاع الكائن كاملًا في JSON يسرّب `path` الداخلي و `buffer`. أعد من الخدمة مفتاحًا عامًا، مثل اسم مولَّد، لا كائن Multer.

## مثال عملي

الخدمة ترفض الغياب وتخزن المفتاح فقط:

```ts
import { BadRequestException, Injectable } from '@nestjs/common';

@Injectable()
export class CoversService {
  replace(bookId: string, file?: Express.Multer.File) {
    if (!file?.filename) {
      throw new BadRequestException('حقل cover مفقود');
    }
    return { bookId, coverKey: file.filename };
  }
}
```

`filename` موجود لأن `diskStorage` سمّى الملف. التخزين في الذاكرة يعطيك `buffer` بلا `filename`. هذا المسار يشترط القرص.

## أخطاء شائعة

- استيراد `FileInterceptor` من `@nestjs/common`. هو في `@nestjs/platform-express`.
- `@UploadedFile()` من غير `@UseInterceptors`، فيصل `undefined`.
- إرجاع `file` كما هو في جسم الرد.
- خلط اسم الحقل: النموذج يرسل `file` والمعترض ينتظر `cover`.

## متى نستخدمه؟

لمسار يستقبل ملفًا واحدًا بحقل معروف: غلاف، إيصال، شعار.

## متى لا نستخدمه؟

لعدة ملفات أو عدة حقول. تلك `FilesInterceptor` و `FileFieldsInterceptor` و `AnyFilesInterceptor`. ولطلب multipart بلا ملفات، `NoFilesInterceptor`.

## مقارنة

| الأداة | دورها على هذا المسار |
| --- | --- |
| `MulterModule.register` | افتراض لكل المعترضات في الوحدة |
| `FileInterceptor(name, options)` | تحليل حقل واحد، وخياراته تغلّب الافتراض |
| `@UploadedFile()` | قراءة الناتج داخل المعالج |

## تمارين

### تمرين سهل

من أي حزمة تستورد `FileInterceptor` على Express؟

<details>
<summary>الحل</summary>
من `@nestjs/platform-express`. أما `@UploadedFile` فمن `@nestjs/common`.
</details>

### تمرين متوسط

النموذج يرسل الحقل `avatar` والمعترض `FileInterceptor('cover')`. ماذا يصل `@UploadedFile()`؟

<details>
<summary>الحل</summary>
لا يصل ملف الغلاف. المعترض يبحث عن جزء باسم `cover` فقط.
</details>

### تمرين متقدم

لماذا إرجاع `file.path` في JSON أسوأ من إرجاع `file.filename`؟

<details>
<summary>الحل</summary>
`path` يكشف مجلد الخادم. `filename` مفتاح ولّدته أنت، وتبني أنت عنوان التحميل لاحقًا إن كان الملف عامًا.
</details>

## أسئلة مراجعة

- ما وسيطا `FileInterceptor`؟
- أي زينة تستخرج الملف بعد التحليل؟
- متى يكون `buffer` موجودًا ومتى يكون `path`؟

## الخلاصة

`FileInterceptor` يعلّق Multer على حقل واحد. `@UploadedFile` يسلّم `Express.Multer.File`. الرد مفتاح تخزين، لا كائن الوسيط.

## ماذا بعد؟

الحقل الواحد لا يكفي لكل النماذج: مصفوفة ملفات، وحقول متعددة، وأي ملف يصل.

## المصادر الرسمية

- NestJS — File upload: https://docs.nestjs.com/http/file-upload
