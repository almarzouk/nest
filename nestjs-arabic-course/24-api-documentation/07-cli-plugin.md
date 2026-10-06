# CLI Plugin

## الهدف من هذا الدرس

تشغيل إضافة `@nestjs/swagger` حتى تُستنتج خصائص DTO والاستجابات من TypeScript ومن `class-validator`، مع الإبقاء على التحقق وقت التشغيل.

## المشكلة التي يحلها

كل حقل في `CreatePostDto` و `PostResponseDto` يحتاج `@ApiProperty` يدويًا. الحد في `@Max(50)` لا يظهر في الوثيقة إلا إن كررته. تعليق فوق `page` لا يصل إلى Swagger. بعد عشرات الحقول ينسى أحدهم مزخرف التوثيق فيختلف النموذج عن الأنبوب.

## الفكرة ببساطة

الإضافة اختيارية وتعمل أثناء الترجمة على شجرة الصياغة. تضع `@ApiProperty` للخصائص، وتستنتج `required` من `?`، وتقرأ قواعد `class-validator` حين يكون `classValidatorShim` مفعّلًا، وهو الافتراضي. تستطيع قراءة التعليقات و `@example` عند `introspectComments`. الملفات التي تُحلَّل تنتهي افتراضيًا بـ `.dto.ts` أو `.entity.ts`.

## التعريف التقني

في `nest-cli.json`:

```json
{
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

مع خيارات:

```json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": true,
          "introspectComments": true
        }
      }
    ]
  }
}
```

بعد تغيير الخيارات احذف `dist` وأعد البناء. الصفحة تذكر ذلك.

ما تفعله الإضافة: تزخرف الخصائص إلا مع `@ApiHideProperty`، تضبط النوع والتعداد والمصفوفات والافتراض، تضيف مزخرف استجابة بحالة و `type`، وتنقل التعليق إلى الوصف عند تفعيل الاستنباط.

`classValidatorShim` الافتراضي `true` ينقل مثلًا `@Max(50)` إلى `max` في المخطط. التعطيل `false` إن أردت وثيقة لا تعكس المحققات. التحقق نفسه لا ينتقل إلى الإضافة. أبقِ `@IsString` و `@Max`. الصفحة تقول ذلك صراحة.

`PartialType` و `PickType` و `OmitType` داخل DTO تُستورد من `@nestjs/swagger` لا من `@nestjs/mapped-types` حتى تلتقط الإضافة المخطط.

`introspectComments` يقرأ وصف الخاصية و `@example`. على المتحكم يحوّل تعليق الطريقة إلى `@ApiOperation` بمفتاح `summary` افتراضيًا. `@param` منذ الإصدار 12 من الحزمة يوثّق `@Query` و `@Param` حسب اسم المتغير لا اسم سلسلة الاستعلام. `@Query('page_size') pageSize` يُوثَّق بـ `@param pageSize`. مزخرف `@ApiQuery` الصريح لا يُستبدل.

`dtoFileNameSuffix` إن سمّيت الملفات غير `.dto.ts`. `classTransformerShim: 'exclusive'` يوثّق فقط `@Expose` أو `@ApiProperty`. `skipAutoHttpCode` يوقف حقن `@HttpCode` الآلي. `esmCompatible` يُكتشف من المشروع. اضبطه إن لزم لمشروع `"type": "module"`.

SWC: الإضافات تحتاج فحص الأنواع، `nest start -b swc --type-check` في التطبيق العادي. المستودع الأحادي يولّد `metadata` ثم `SwaggerModule.loadPluginMetadata`. التفاصيل في الصفحة.

`ts-jest` لا يشغّل إضافة Nest وحده. صفحة CLI تشرح محوّل AST للاختبارات e2e ومسح الكاش عند تجاهل Jest للإعداد.

مزخرف صريح يغلب الاستنتاج. أبقِ `@ApiProperty` حيث المثال أدق من التعليق.

## Mental Model

```text
nest build يقرأ .dto.ts
  يستنتج الخصائص والحدود
  class-validator يبقى عند الطلب
تغيير الخيارات يحتاج مسح dist
```

## أبسط مثال

```ts
export class ListPostsQueryDto {
  /** Page number starting at 1. @example 1 */
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page = 1;
}
```

## شرح المثال

مع `introspectComments` و shim يصبح للخاصية وصف ومثال وحد أدنى في الوثيقة. `@Type` ما زال للتحويل. `@IsInt` ما زال للرفض. اسم الملف `list-posts.query.dto.ts` يطابق اللاحقة. ملف `posts.types.ts` لا تلمسه الإضافة إلا إن وسّعت `dtoFileNameSuffix`.

## ماذا يحدث داخليًا؟

المحوّل يحقن المزخرفات في الخرج المترجم. وقت التشغيل يرى البيانات الوصفية كأنها كُتبت يدويًا. `createDocument` لا يعرف الفرق. لذلك وثيقة `nest start` تختلف عن تشغيل ملف بلا مرور على المحوّل، مثل اختبار Jest بلا إعداد الصفحة.

تعليق `@example` الخاطئ لا يغيّر قاعدة البيانات. يغيّر ما يملؤه المجرب في Swagger.

## مثال عملي

فعّل الإضافة ثم احذف `@ApiProperty` من `CreatePostDto` مؤقتًا وأبقِ المحققات. أعد البناء وافتح `/api`. يجب أن يظهر `title`. أعد المزخرف إن احتجت مثالًا أغنى من التعليق. حوّل `UpdatePostDto` إلى `PartialType(CreatePostDto)` مستوردًا من `@nestjs/swagger` وتأكد أن `title` اختياري في `PATCH`.

على SWC، إن بقيت النماذج فارغة، شغّل مع `--type-check` أو حمّل metadata كما في الصفحة قبل أن تعيد المزخرفات كلها يدويًا.

## أخطاء شائعة

- حذف `@IsString` لأن الوثيقة تعرض النوع.
- `PartialType` من الحزمة الأخرى فتفقد الإضافة الحقول.
- نسيان مسح `dist` بعد تغيير `introspectComments`.
- توقع أن Jest e2e يرى الإضافة بلا محوّل.

## متى نستخدمه؟

في مشروع تكثر فيه DTO وتريد الوثيقة أن تتبع الأنواع والمحققات.

## متى لا نستخدمه؟

كبديل عن مراجعة أمن الوثيقة. الإضافة قد تنشر حقلًا في كيان `.entity.ts` فيه `passwordHash` إن كان الصنف مذكورًا. أخفِه بـ `@ApiHideProperty` أو لا تمرّر الكيان كـ `type` للاستجابة.

## مقارنة

| مزخرف يدوي | إضافة CLI |
| --- | --- |
| صريح في المصدر | يُحقن عند الترجمة |
| يعمل بلا nest-cli | يحتاج البناء أو metadata |
| مثال تكتبه في الكود | تعليق `@example` عند التفعيل |

## تمارين

### تمرين سهل

ما لاحقة الملف التي تراها الإضافة افتراضيًا؟

<details>
<summary>الحل</summary>

`.dto.ts` و `.entity.ts`.

</details>

### تمرين متوسط

من أين تستورد `PartialType` مع الإضافة؟

<details>
<summary>الحل</summary>

من `@nestjs/swagger`.

</details>

### تمرين متقدم

لماذا تبقى النماذج فارغة تحت `nest start -b swc` بلا فحص أنواع؟

<details>
<summary>الحل</summary>

إضافة CLI لا تعمل على بناء SWC العادي من غير `--type-check` أو `loadPluginMetadata` في إعداد المستودع الأحادي.

</details>

## أسئلة مراجعة

- هل الإضافة تستبدل `ValidationPipe`؟
- ماذا يفعل `classValidatorShim`؟
- متى تمسح `dist`؟

## الخلاصة

الإضافة تملأ الوثيقة من الملفات ذات اللاحقة الصحيحة ومن المحققات والتعليقات. الأنبوب يبقى. لا تدع كيان كلمة المرور يصبح نموذج الاستجابة لأن الملف ينتهي بـ `.entity.ts`.

## ماذا بعد؟

[أمثلة عملية](./examples.md) وثيقة المقالات مجمّعة.

## المصادر الرسمية

- NestJS — CLI plugin: https://docs.nestjs.com/openapi/cli-plugin
