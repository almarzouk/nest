# الحجم و MIME Type

## الهدف من هذا الدرس

رفض ملف الغلاف قبل حفظه في المجال: سقف بايتات، ونوع محتوى يُستنتج من البايتات لا من اسم الملف.

## المشكلة التي يحلها

`diskStorage` يحفظ ما وصله ما دام تحت حد Multer. العميل يسمّي `cover.exe` إلى `cover.png` ويضع `Content-Type: image/png`. الامتداد الذي ألصقته في `filename` نسخة من هذا الادعاء. المجال يخزّن مفتاحًا سيعامَل لاحقًا كصورة.

## الفكرة ببساطة

حدّان متتاليان. `limits.fileSize` في Multer يقطع القراءة. ثم `ParseFilePipe` يشغّل `MaxFileSizeValidator` و `FileTypeValidator` على الملف الذي نجا. `FileTypeValidator` يطابق النوع من الرقم السحري في المحتوى افتراضيًا، لا من الترويسة وحدها.

## التعريف التقني

الأنبوب المدمج `ParseFilePipe` يأخذ `validators`. كل مدقق يطبّق `FileValidator`: `isValid` و `buildErrorMessage`. المدققان الجاهزان:

- `MaxFileSizeValidator({ maxSize })` بالبايت.
- `FileTypeValidator({ fileType })` نص أو تعبير. الافتراضي فحص magic number. `skipMagicNumbersValidation: true` يقارن النوع الذي أعلنه العميل فقط، فتجنّبه للأغلفة.

الخيار `errorHttpStatusCode` افتراضه `400`. `exceptionFactory` إن أردت شكل خطأ واحدًا مع بقية API. الملف مطلوب افتراضيًا. لجعله اختياريًا: `fileIsRequired: false` داخل `build()` في `ParseFilePipeBuilder`.

الأنبوب يُربَط على `@UploadedFile(...)`. مع المصفوفات، الأنابيب على `@UploadedFiles()` تجري لكل عنصر.

## Mental Model

```text
Multer limits.fileSize     يوقف التيار المبكر
ParseFilePipe
  MaxFileSizeValidator     سقف ثانٍ للعقد (بايت)
  FileTypeValidator        magic number للصورة
الخدمة                     لا ترى إلا ما عبر
```

الأنبوب يعمل بعد اكتمال قبضة Multer. لذلك السقف الكبير يبقى على `limits` حتى لا يُكتب ملف ضخم ثم يُحذف.

## أبسط مثال

```ts
import {
  FileTypeValidator,
  MaxFileSizeValidator,
  ParseFilePipe,
} from '@nestjs/common';

export const coverPipe = new ParseFilePipe({
  validators: [
    new MaxFileSizeValidator({ maxSize: 2 * 1024 * 1024 }),
    new FileTypeValidator({ fileType: /^image\/(png|jpeg)$/ }),
  ],
});
```

على المسار:

```ts
@Post(':id/cover')
@UseInterceptors(FileInterceptor('cover', coverLimits))
replace(
  @Param('id') bookId: string,
  @UploadedFile(coverPipe) file: Express.Multer.File,
) {
  return this.covers.replace(bookId, file.filename);
}
```

`coverPipe` يفترض وجود الملف. غيابه فشل تحقق، لا فرع `undefined`.

## شرح المثال

التعبير يقبل PNG و JPEG كما يستنتجها الفاحص من المحتوى. `image/png` الذي كتبه المتصفح لا يكفي وحده. `maxSize` هنا يساوي حد Multer حتى تبقى رسالة العقد واحدة. إن أردت `422` بدل `400`:

```ts
new ParseFilePipeBuilder()
  .addMaxSizeValidator({ maxSize: 2 * 1024 * 1024 })
  .addFileTypeValidator({ fileType: /^image\/(png|jpeg)$/ })
  .build({ errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY });
```

## ماذا يحدث داخليًا؟

بعد التخزين على القرص، `FileTypeValidator` يقرأ الرقم السحري من بداية الملف. عدم التطابق يرمي استثناء قبل الخدمة، لكن `diskStorage` قد يكون كتب الملف. انقل من مجلد مؤقت بعد النجاح، أو احذف اليتيم. الرقم السحري يرفض التنكر الشائع، ولا يثبت أن الصورة سليمة حتى آخر بايت.

## مثال عملي

سياسة غلاف المتجر:

| القاعدة | الأداة |
| --- | --- |
| لا أكبر من 2 ميغابايت | `limits.fileSize` ثم `MaxFileSizeValidator` |
| PNG أو JPEG فقط | `FileTypeValidator` بالرقم السحري |
| الاسم على القرص | `randomUUID` من درس Multer |
| المفتاح في الجدول | `filename` بعد نجاح الأنبوب |

عينة PDF على حقل `sample` تستخدم أنبوبًا ثانيًا بتعبير `application/pdf`. لا تعِد استعمال `coverPipe` على الحقلين.

## أخطاء شائعة

- قبول الملف لأن الامتداد `.png`.
- `skipMagicNumbersValidation: true` في مسار عام.
- سقف الأنبوب 2 ميغابايت وسقف Multer غائب، فيُخزَّن ملف أكبر ثم يُرفض.
- ترك ملف القرص بعد فشل الأنبوب بلا سياسة حذف.

## متى نستخدمه؟

على كل مسار رفع يغيّر مجالًا: غلاف، مرفق، شعار. المدققات في ثابت مسمّى إن طالت القائمة، كما تقترح الصفحة.

## متى لا نستخدمه؟

كبديل عن حد Multer. وكفحص وحيد لملف سيُنفَّذ أو يُفك ضغطه. ذلك مسار عزل مختلف عن مطابقة الصورة.

## مقارنة

| الإشارة | من يكتبها | قوتها |
| --- | --- | --- |
| الامتداد | العميل | ضعيفة |
| `mimetype` في الجزء | العميل | ضعيفة |
| magic number | `FileTypeValidator` | مناسبة للصورة الشائعة |
| `limits.fileSize` | Multer أثناء القراءة | تمنع التخزين الزائد |

## تمارين

### تمرين سهل

ما وحدة `maxSize` في `MaxFileSizeValidator`؟

<details>
<summary>الحل</summary>
البايت. `2 * 1024 * 1024` يعني 2 ميغابايت.
</details>

### تمرين متوسط

لماذا لا يكفي أن يطابق `file.mimetype` القيمة `image/png`؟

<details>
<summary>الحل</summary>
لأن العميل يضع الترويسة. `FileTypeValidator` ينظر إلى محتوى الملف ما لم تُعطّل فحص الرقم السحري.
</details>

### تمرين متقدم

نجح Multer في الكتابة ثم فشل `FileTypeValidator`. أين الملف، وما الذي لم يحدث؟

<details>
<summary>الحل</summary>
الملف على المسار الذي اختاره `diskStorage`. الخدمة لم تُستدعَ. تحتاج حذفًا أو كتابة مؤقتة ثم نقلًا بعد نجاح الأنبوب.
</details>

## أسئلة مراجعة

- أين يُربَط `ParseFilePipe`؟
- ماذا يفعل `fileIsRequired: false`؟
- لماذا يبقى `limits.fileSize` حتى مع وجود `MaxFileSizeValidator`؟

## الخلاصة

الحجم يُقطع في Multer ويُعاد التحقق في الأنبوب. النوع للصورة يُستنتج من البايتات. الامتداد ادّعاء عميل.

## ماذا بعد؟

إرسال ملف إلى العميل بتيار، من غير تحميله كله في الذاكرة ومن غير إغلاق باب المعترضات.

## المصادر الرسمية

- NestJS — File upload: https://docs.nestjs.com/http/file-upload
