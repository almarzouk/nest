# إرسال الملفات إلى العميل

## الهدف من هذا الدرس

إرجاع غلاف أو تقرير من مسار HTTP عبر `StreamableFile`، بحيث تُضخ البايتات إلى العميل وتبقى المعترضات بعد المتحكم قادرة على العمل.

## المشكلة التي يحلها

`fs.readFile` يحمّل الملف كاملًا في `Buffer` ثم يضعه في الرد. تقرير كبير أو غلاف يُطلب كثيرًا يرفع الذاكرة مع كل طلب. `file.pipe(res)` مع `@Res()` يرسل التيار، لكنه يسلب Nest الرد: منطق المعترضات بعد المتحكم لا يجري.

## الفكرة ببساطة

افتح `createReadStream` وأعد `new StreamableFile(stream)`. Nest تضخ التيار في الرد على Express و Fastify. النوع والاسم في خيارات المنشئ أو عبر `@Header` أو `@Res({ passthrough: true })` إن كانت الترويسة محسوبة وقت التشغيل.

## التعريف التقني

`StreamableFile` من `@nestjs/common`. المنشئ يقبل `Readable` أو `Uint8Array`. تمرير `Buffer` يعني أن المحتوى محبوس مسبقًا. التوثيق يخص هذا الأسلوب بتطبيقات HTTP، لا GraphQL ولا الخدمات المصغرة.

خيارات المنشئ:

| الخيار | الأثر |
| --- | --- |
| `type` | `Content-Type`، والافتراض `application/octet-stream` |
| `disposition` | `Content-Disposition`، مثل إرفاق باسم ملف |
| `length` | `Content-Length` إن عرفته مسبقًا للتيار |

مسلسلات الصنف لا تسلسل `StreamableFile`. على Express، خطأ التيار قبل أي بايت يصبح `400`، و `setErrorHandler` يبدّل ذلك. على Fastify الافتراض `500` عبر طبقة الاستثناءات.

## Mental Model

```text
createReadStream(path)
  StreamableFile
    Nest تكتب الترويسات
    ثم تضخ الكتل
المعترض بعد المتحكم ما زال يرى القيمة المعادة
```

`@Res()` بلا `passthrough` فرع آخر: أنت تملك `res`، والإطار لا يكمل الإرسال نيابة عنك.

## أبسط مثال

```ts
import { Controller, Get, Header, Param, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'node:fs';
import { CoversService } from './covers.service.js';

@Controller('books')
export class CoverFilesController {
  constructor(private readonly covers: CoversService) {}

  @Get(':id/cover')
  @Header('Cache-Control', 'private, max-age=3600')
  download(@Param('id') bookId: string): StreamableFile {
    const stored = this.covers.resolve(bookId);
    const stream = createReadStream(stored.absolutePath);
    return new StreamableFile(stream, {
      type: stored.mime,
      disposition: `inline; filename="${stored.downloadName}"`,
      length: stored.bytes,
    });
  }
}
```

`resolve` تُرجع مسارًا مطلقًا بنته الخدمة من مفتاح مخزّن، لا من سلسلة أرسلها العميل في الاستعلام.

## شرح المثال

`inline` يعرض الصورة في المتصفح. `attachment` ينزّلها. `length` معروف لأننا حفظنا الحجم عند الرفع، فيقدر العميل النسبة. `Cache-Control: private` لأن الغلاف قد يكون لمستخدم مسجّل. المسار الثابت العام، إن وُجد، عمل `ServeStaticModule` في الفصل التالي لا هذا المتحكم.

## ماذا يحدث داخليًا؟

`createReadStream` لا يقرأ الملف عند بناء الكائن. القراءة تبدأ عندما تسحب Nest التيار. `setErrorHandler` يحوّل `ENOENT` قبل إرسال البايتات إلى `404` إن لم تُرسل الترويسات. لا تمرر `bookId` إلى `join` من العميل: `../` يهرب من المجلد. الخدمة تترجم المعرف إلى اسم `randomUUID`.

## مثال عملي

تقرير جاهز يُرسل بـ `StreamableFile` ونوع `application/pdf` و `disposition: attachment`. `pathFor` تقبل أسماءً من قائمة بيضاء. البناء الثقيل لا يحدث داخل المسار. `file.buffer` للرفع الكبير و `readFile` للتنزيل الكبير كلاهما يحبس الملف. الحجم الكبير يريد قرصًا أو تيارًا.

## أخطاء شائعة

- `readFileSync` ثم `new StreamableFile(buffer)` لملف كبير، فتختفي فائدة التيار.
- `@Res()` ثم `stream.pipe(res)` فتتوقف معترضات القياس بعد المتحكم.
- أخذ مسار الملف من `?path=` الذي يرسله العميل.
- نسيان `type` فيصل `application/octet-stream` لصورة PNG.

## متى نستخدمه؟

عندما يكون الملف خلف قاعدة: صلاحية، اسم منطقي، أو ترويسة تُحسب لكل طلب.

## متى لا نستخدمه؟

لملفات عامة ثابتة بلا تفويض. تلك جذر `ServeStaticModule`. ولرفع كبير على Express، هذا الصنف لا يحوّل القبضة إلى تيار وارد. الوارد الكبير يبقى `diskStorage` مع حد، أو `FileStreamInterceptor` على Fastify.

## مقارنة

| الأسلوب | الذاكرة | المعترض بعد المتحكم |
| --- | --- | --- |
| `readFile` ثم JSON أو Buffer | الملف كله | يعمل |
| `@Res()` و `pipe` | كتل | يتوقف لأنك تملك الرد |
| `StreamableFile` | كتل | يعمل |

## تمارين

### تمرين سهل

من أي حزمة تُستورد `StreamableFile`؟

<details>
<summary>الحل</summary>

من `@nestjs/common`.

</details>

### تمرين متوسط

لماذا `passthrough: true` عند ضبط ترويسة ديناميكية؟

<details>
<summary>الحل</summary>

حتى تكتب الترويسة على `res` وتُرجع `StreamableFile` فيكمل Nest الإرسال. من غير `passthrough` يصبح الرد ملك المعالج.

</details>

### تمرين متقدم

الغلاف يُطلب بمعرف كتاب. لماذا لا تقرأ `join(uploadRoot, id)` مباشرة؟

<details>
<summary>الحل</summary>

المعرف قد يحمل مسارًا نسبيًا. الخدمة تترجم المعرف إلى اسم مولَّد خُزّن عند الرفع، ثم تبني المسار المطلق من ذلك الاسم.

</details>

## أسئلة مراجعة

- متى تضع `length`؟
- ماذا يفعل `setErrorHandler` على Express قبل إرسال البايتات؟
- أي اتجاه يبقى خطرًا إن حُبس في `Buffer`: الرفع الكبير أم التنزيل الكبير؟

## الخلاصة

`StreamableFile` فوق `createReadStream` يرسل الملف كتلاً ويبقي الرد داخل Nest. المسار يأتي من مفتاح خزّنته، لا من سلسلة العميل.

## ماذا بعد؟

الملفات العامة التي تُخدم من مجلد بلا متحكم لكل صورة، ومتى يبتلع ذلك مسارات الـ API.

## المصادر الرسمية

- NestJS — Streaming files: https://docs.nestjs.com/http/file-upload
