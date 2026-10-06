# Multer

## الهدف من هذا الدرس

معرفة ما يفعله Multer بجسم `multipart/form-data` على Express: أين تُحبس البايتات، وكيف تُرفض قبل المعالج.

## المشكلة التي يحلها

بعد أن يصل الطلب، شيء ما يجب أن يفصل الحقول عن الملف. إن حُبس ملف كبير في `file.buffer` امتلأت ذاكرة العملية مع طلبات متوازية. Multer هو هذا المحلل في مسار Express، وسلوكه الافتراضي ذاكرة لا قرص.

## الفكرة ببساطة

Multer وسيط Express يفهم multipart. Nest لا تعيد تنفيذ التحليل. هي تمرر خيارات Multer عبر `MulterModule` أو عبر المعترض. من غير `dest` ومن غير `storage` يبقى الملف في الذاكرة. للملفات الكبيرة اختر تخزين قرص وحد حجم.

## التعريف التقني

التوثيق: الملفات تبقى في الذاكرة داخل `file.buffer` ما لم تضبط `dest` أو `storage`. `dest` يكتب كل ملف في مجلد باسم عشوائي. `diskStorage()` يتيح لك المجلد واسم الملف، وعندها تمتلئ `destination` و `filename` و `path` ويغيب `buffer`.

خيارات تكثر الحاجة إليها:

| الخيار | الأثر |
| --- | --- |
| `limits.fileSize` | يقطع الجزء إذا تجاوز البايتات المحددة |
| `limits.files` | أقصى عدد ملفات في الطلب |
| `storage` | محرك التخزين، مثل `diskStorage` من حزمة `multer` |
| `fileFilter` | قرار قبول أو رفض حسب وصف الملف، قبل اكتمال القصة |

التسجيل الافتراضي على الوحدة:

```ts
import { Module } from '@nestjs/common';
import { MulterModule } from '@nestjs/platform-express';
import { diskStorage } from 'multer';

@Module({
  imports: [
    MulterModule.register({
      storage: diskStorage({
        destination: './uploads',
      }),
      limits: { fileSize: 2 * 1024 * 1024 },
    }),
  ],
})
export class CoversModule {}
```

`MulterModule` من `@nestjs/platform-express`. الإعداد غير المتزامن `registerAsync` مع `useFactory` و `inject`. على Fastify المحلل `@fastify/multipart` والمعترضات من `@nestjs/platform-fastify/multipart`. هذا الدرس يبقى على Express.

## Mental Model

```text
تيار multipart
  Multer يقرأ جزءًا جزءًا
    حد fileSize انكسر  -> رفض قبل handler
    بلا storage        -> buffer في الذاكرة
    diskStorage        -> كتابة على المسار، والذاكرة لا تحتفظ بالملف
```

الحد هنا أرخص من فحص لاحق، لأن الفحص بالأنبوب يجري بعد أن ينتهي Multer من الإمساك بالملف.

## أبسط مثال

محرك قرص يرفض الاسم الذي يأتي من العميل كمسار.

```ts
import { randomUUID } from 'node:crypto';
import { extname } from 'node:path';
import { diskStorage } from 'multer';

export const coverStorage = diskStorage({
  destination: './uploads/covers',
  filename: (_req, file, callback) => {
    const extension = extname(file.originalname).toLowerCase();
    callback(null, `${randomUUID()}${extension}`);
  },
});
```

الامتداد هنا تلميح للاسم فقط. التحقق من المحتوى درس لاحق. `originalname` لا يدخل `destination`.

## شرح المثال

`callback(null, name)` يعني نجاح التسمية. `randomUUID` يمنع تصادم الأسماء ويمنع `../` في اسم المتصفح. المجلد النسبي يتبع مجلد تشغيل العملية، فثبّته بمسار مطلق في التطبيق الحقيقي.

## ماذا يحدث داخليًا؟

Multer يسجّل نفسه كوسيط على المسار عندما يركّب Nest المعترض. أثناء القراءة يحصي البايتات. إذا تجاوز `limits.fileSize` يتوقف ويُرجع خطأ طلب كبير، ولا يُستدعى المعالج. مع التخزين في الذاكرة تتجمع البايتات في `Buffer` واحد لكل ملف. مع القرص تُكتب الكتل إلى ملف مؤقت أو نهائي، والكائن الذي يصل المعالج يحمل `path` لا `buffer`. عدة طلبات كبيرة في الذاكرة ترفع RSS معًا. القرص يبدل الضغط إلى I/O ومساحة المجلد.

## مثال عملي

غلاف حتى 2 ميغابايت على القرص، ورفض ما هو أكبر قبل الخدمة:

```ts
export const coverLimits = {
  storage: coverStorage,
  limits: { fileSize: 2 * 1024 * 1024, files: 1 },
};
```
مرّر `coverLimits` إلى `FileInterceptor('cover', coverLimits)` في الدرس التالي. الخدمة تستلم مسارًا جاهزًا، ولا تستدعي `fs.readFile` على الملف كله لتتأكد أنه «وصل».

## أخطاء شائعة

- الاعتماد على الافتراضي ثم استغراب `file.buffer` لملف بمئات الميغابايت.
- حفظ `file.originalname` داخل `join(destination, originalname)`.
- ضبط الحد في الأنبوب فقط، بعد أن يكون الملف قد سكن الذاكرة.
- مجلد `uploads` داخل جذر يخدمه `ServeStaticModule` بلا تمييز بين العام والخاص.

## متى نستخدمه؟

على تطبيق Express كلما كان العقد `multipart/form-data`. اضبط `storage` و `limits` في `MulterModule.register` إن كانت السياسة واحدة لكل المسارات.

## متى لا نستخدمه؟

على Fastify: الحزمة هناك `@fastify/multipart` لا Multer. ولرفع ضخم يُضخ إلى مخزن وأثناء وصوله، Express مع `diskStorage` ما زال يمسك الملف حتى يكتمل الجزء. الضخ أثناء الوصول في التوثيق هو `FileStreamInterceptor` على Fastify.

## مقارنة

| الاختيار | أين البايتات قبل المعالج | متى يليق |
| --- | --- | --- |
| بلا `storage` | `file.buffer` | ملف صغير معروف السقف |
| `dest: './uploads'` | قرص باسم عشوائي | تريد إبعاد الذاكرة فورًا |
| `diskStorage` | قرص ومسار تحدده | اسم مولَّد ومجلد مجال |

## تمارين

### تمرين سهل

أين تعيش بايتات الملف إن لم تضبط `dest` ولا `storage`؟

<details>
<summary>الحل</summary>
في الذاكرة، على `file.buffer`.
</details>

### تمرين متوسط

لماذا `limits.fileSize` على Multer أهم من فحص الحجم داخل الخدمة؟

<details>
<summary>الحل</summary>
لأن Multer يرفض أثناء القراءة. الخدمة لا تعمل إلا بعد أن أمسك المحلل الملف كاملًا.
</details>

### تمرين متقدم

`filename` يعيد `file.originalname` كما وصل. ما الخطر مع قرص وطلبين؟

<details>
<summary>الحل</summary>
اسم العميل قد يحتوي `../` أو يتصادم فيُكتب ملف فوق آخر. ولّد الاسم في الخادم.
</details>

## أسئلة مراجعة

- ما الحزمة التي تصدّر `MulterModule` على Express؟
- ماذا يملأ `diskStorage` بدل `buffer`؟
- لماذا المجلد النسبي `./uploads` قرار بيئة لا قرار زينة؟

## الخلاصة

Multer يحلّل multipart على Express. الافتراضي ذاكرة. الملفات الكبيرة تذهب إلى قرص مع حد بايتات، وباسم يولّده الخادم.

## ماذا بعد؟

`FileInterceptor`: كيف تُعلّق خيارات Multer هذه على مسار واحد وتستلم الملف في المعامل.

## المصادر الرسمية

- NestJS — File upload: https://docs.nestjs.com/http/file-upload
