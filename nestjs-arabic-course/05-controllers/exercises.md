# تمارين الفصل

أجب من غير فتح الدروس، ثم افتح الحل.

## سهل

1. ما رمز النجاح الافتراضي لـ `GET` ولـ `POST` عندما يعيد المعالج كائنًا؟
2. ماذا تفعل الوسيطة النصية في `@Controller('holds')`؟

<details>
<summary>الحل</summary>

1. `GET` يستخدم 200، و `POST` يستخدم 201، ما لم يُضبط `@HttpCode`.
2. تثبت بادئة `/holds` لكل معالجات الصنف. نص `@Get` أو غيره يُلحَق بها.

</details>

## متوسط

صمم توقيع `POST /holds` لإنشاء حجز، و `GET /holds/:id` لقراءته، و `DELETE /holds/:id` لإلغائه بلا جسم. العميل يرسل `days` في JSON، و `Idempotency-Key` في رأس، وقد يمرر `?notify=mail`. اذكر الـ Decorator لكل مدخل والرمز الذي تريده للإلغاء.

<details>
<summary>الحل</summary>

- الإنشاء: `@Post()` و `@Body() dto` فيه `days`، و `@Headers('idempotency-key')` بالأحرف الصغيرة. الرمز الافتراضي 201 يناسب إنشاء مورد.
- القراءة: `@Get(':id')` و `@Param('id') id: string`.
- الإشعار قناة اختيارية: `@Query('notify')` على الإنشاء أو على مسار خاص، والقيمة نص.
- الإلغاء: `@Delete(':id')` و `@Param('id')` و `@HttpCode(204)` إن كان العقد بلا جسم.
- ضع أي مسار ثابت، إن أضفته لاحقًا، قبل `@Get(':id')` على Express الافتراضي.

</details>

## متقدم

ثلاثة مسارات في Controller واحد تعطلت. شخّص كل عرض:

1. `GET /holds/me` يعيد حجزًا معرفه `me`.
2. `POST /holds` يعيد `{ id }` من الدالة، والعميل ينتظر حتى انتهاء المهلة، والتوقيع فيه `@Res() res`.
3. `page` في `GET /holds?page=2` يصبح `"21"` بعد `page + 1` داخل الخدمة.

<details>
<summary>الحل</summary>

1. `@Get(':id')` مُسجَّل قبل `@Get('me')` والاستراتيجية `declaration` على Express. `me` يطابق المعامل. انقل الثابت قبل النمط أو فعّل `routeResolutionStrategy: 'specificity'`.
2. `@Res()` بلا `passthrough` يوقف الإرسال القياسي. `return { id }` لا يُرسل، ولم يُستدعَ `res.send`. إما `res.json` في وضع المكتبة، أو احذف `@Res()` وأعد الكائن، أو استخدم `passthrough: true` إن كنت تحتاج لمس الكائن فقط.
3. `@Query` يمرر نصًا. `"2" + 1` يلصق النصين. حوّل تحويلًا ظاهرًا أو انتظر Pipe في فصل لاحق. تعليق `number` لا يغيّر القيمة.

</details>
