# أمثلة عملية

ثلاثة مسارات على متجر كتب. الافتراض Express، وتخزين القرص عبر `diskStorage`، والاستيراد المحلي ينتهي بـ `.js` كما في قالب ESM.

## مثال 1 — غلاف على القرص وبسقف

`covers.storage.ts` يولّد الاسم. المعالج لا يرى `originalname` إلا للتشخيص.

```ts
import { randomUUID } from 'node:crypto';
import { extname } from 'node:path';
import { diskStorage } from 'multer';
import type { MulterOptions } from '@nestjs/platform-express';

export const coverUpload: MulterOptions = {
  storage: diskStorage({
    destination: './var/covers',
    filename: (_req, file, cb) => {
      cb(null, `${randomUUID()}${extname(file.originalname).toLowerCase()}`);
    },
  }),
  limits: { fileSize: 2 * 1024 * 1024, files: 1 },
};
```

```ts
import { FileTypeValidator, MaxFileSizeValidator, ParseFilePipe } from '@nestjs/common';

export const coverPipe = new ParseFilePipe({
  validators: [
    new MaxFileSizeValidator({ maxSize: 2 * 1024 * 1024 }),
    new FileTypeValidator({ fileType: /^image\/(png|jpeg)$/ }),
  ],
});
```

المتحكم يمرر المفتاح إلى الخدمة. الخدمة تحدّث عمود `coverKey` ولا تخزن `Express.Multer.File`.

```ts
@Post(':id/cover')
@UseInterceptors(FileInterceptor('cover', coverUpload))
replace(@Param('id') id: string, @UploadedFile(coverPipe) file: Express.Multer.File) {
  return this.books.setCoverKey(id, file.filename);
}
```

جرب: `curl -F "cover=@./a.png;type=image/png" localhost:3000/books/1/cover`. ثم أعد الطلب بملف أكبر من 2 ميغابايت وتوقع الرفض قبل تحديث الصف.

## مثال 2 — غلاف إجباري وعينة PDF

حقلان، أنبوبان. العينة اختيارية عبر `fileIsRequired: false`.

```ts
const samplePipe = new ParseFilePipeBuilder()
  .addMaxSizeValidator({ maxSize: 8 * 1024 * 1024 })
  .addFileTypeValidator({ fileType: 'application/pdf' })
  .build({ fileIsRequired: false });

@Post()
@UseInterceptors(FileFieldsInterceptor([
  { name: 'cover', maxCount: 1 },
  { name: 'sample', maxCount: 1 },
], coverUpload))
create(
  @Body() body: CreateBookDto,
  @UploadedFiles() files: { cover?: Express.Multer.File[]; sample?: Express.Multer.File[] },
) {
  const cover = files.cover?.[0];
  if (!cover) throw new BadRequestException('cover is required');
  return this.books.create(body, {
    coverKey: cover.filename,
    sampleKey: files.sample?.[0]?.filename,
  });
}
```

`ParseFilePipe` على مصفوفة الحقول في التوثيق يمشي على كل ملف. إن ربطت أنبوب الغلاف على الكائن كله سيُرفض PDF. افصل المدققات حسب الحقل داخل الخدمة إن صعب تركيب أنبوبين على خريطة واحدة، وابقِ حد Multer على الطلب كله.

## مثال 3 — تنزيل الغلاف بتيار

```ts
@Get(':id/cover')
download(@Param('id') id: string): StreamableFile {
  const stored = this.covers.resolve(id);
  return new StreamableFile(createReadStream(stored.absolutePath), {
    type: stored.mime,
    length: stored.bytes,
    disposition: `inline; filename="${stored.downloadName}"`,
  });
}
```

`resolve` ترمي `NotFoundException` إن لم يكن للكتاب `coverKey`. لا تبنِ المسار من `id` الخام.

## ماذا تتأكد منه

- طلب JSON إلى مسار الغلاف يفشل لأن Multer ينتظر multipart.
- هذه الأمثلة لا تقرأ `file.buffer`. المحرك قرص.
- الرد لا يحتوي المسار المطلق.
- `var/covers` خارج جذر الملفات العامة في الفصل التالي.
- بعد فشل `FileTypeValidator` ابحث عن ملف يتيم في `var/covers` بلا صف مقابل.

## أوامر التجربة

```bash
curl -F "cover=@./cover.png;type=image/png" http://localhost:3000/books/1/cover
curl -D - http://localhost:3000/books/1/cover -o /tmp/cover.out
```

الأول يعيد `{ "coverKey": "<uuid>.png" }` إن عبر الحجم والنوع. الثاني يرسل `Content-Type` الصورة و `Content-Length` إن خزّنت `bytes` عند الرفع. غيّر الملف إلى `.txt` مع ترويسة `image/png` وتوقع رفض `FileTypeValidator` لأن الرقم السحري لا يطابق الصورة.
