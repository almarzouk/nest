# تمارين الفصل

أجب من غير فتح الدروس. المجال متجر كتب على Express.

## سهل

1. أي `Content-Type` يرسله نموذج فيه ملف؟
2. أين تذهب البايتات إن لم تضبط `storage` ولا `dest`؟
3. أي زينة تستخرج ملف حقل واحد؟

<details>
<summary>الحل</summary>

1. `multipart/form-data` مع `boundary`.
2. إلى `file.buffer` في الذاكرة.
3. `@UploadedFile()` بعد `FileInterceptor`.

</details>

## متوسط

صمم `POST /books/:id/cover` و `GET /books/:id/cover`.

اذكر: اسم الحقل، خيارات Multer، الأنبوب، ماذا يُحفظ في الجدول، وماذا يُعاد عند التنزيل.

<details>
<summary>الحل</summary>

- الحقل `cover` مع `FileInterceptor` و `diskStorage` باسم `randomUUID`.
- `limits.fileSize` عند 2 ميغابايت مثلًا.
- `ParseFilePipe` فيه `MaxFileSizeValidator` و `FileTypeValidator` لـ PNG و JPEG بالرقم السحري.
- الجدول يحفظ `filename` فقط.
- `GET` يعيد `StreamableFile` من `createReadStream` على مسار بنته الخدمة من المفتاح، مع `type` و `length`.

</details>

## متقدم

نموذج واحد ينشئ كتابًا: `title` نص، `cover` صورة واحدة، `photos` حتى أربع صور، وملف خامس باسم يختاره العميل. ارفض التصميم الأخير، وضع كل معترض في مكانه، واشرح لماذا فشل الأنبوب بعد الكتابة على القرص ما زال يترك ملفًا.

<details>
<summary>الحل</summary>

- `FileFieldsInterceptor` أو معترضات لا تُكدس بسهولة على الدالة نفسها. الشكل الأنسب حقول معلنة: `cover` بحد 1 و `photos` عبر حقل واحد. عمليًا مساران أوضح: `POST /books` للنص، ثم `POST /books/:id/cover` و `POST /books/:id/photos` مع `FilesInterceptor('photos', 4)`.
- الحقل ذو الاسم الحر يُرفض: لا `AnyFilesInterceptor` في هذا العقد.
- `diskStorage` يكتب قبل `ParseFilePipe`. الفشل يترك ملفًا بلا صف. انقل من مجلد مؤقت بعد النجاح، أو احذف `filename` في مسار الخطأ.
- لا تخزن `Express.Multer.File` ولا كائن Prisma داخل أي رد أو مهمة. المفتاح نص.

</details>

## سؤال ربط

ارسم تسلسل طلب `POST` لغلاف حجمه 5 ميغابايت والحد 2 ميغابايت. سمِّ المحطة التي تتوقف عندها، وما الذي لا يتغير في جدول `Book`.

<details>
<summary>الحل</summary>

Multer يقرأ الجزء حتى `limits.fileSize` ثم يرفض. `ParseFilePipe` لا يعمل، و `BooksService.setCoverKey` لا يُستدعى. عمود `coverKey` يبقى على قيمته السابقة. لا يُكتب ملف نهائي باسم UUID إن قطع المحرك الكتابة مع الخطأ. إن كان محركك يترك ملفًا جزئيًا فنظّفه، لكن الصف لا يُحدَّث في الحالتين.

</details>
