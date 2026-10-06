# ما هو Controller

## الهدف من هذا الدرس

تعريف Controller: الصنف الذي يستقبل طلب HTTP ويعيد قيمة، ويُبقي قاعدة العمل في Service.

## المشكلة التي يحلها

في Express تختلط قراءة الطلب مع SQL وشكل الرد داخل دالة واحدة. تغيير قائمة الرفوف يعدّل الدالة التي تعرف أيضًا رمز الحالة. Controller يفصل الاستقبال عن القرار.

## الفكرة ببساطة

Controller صنف عليه Decorator. Nest تجمع من الـ Decorators خريطة تربط الطلب بمعالج. المعالج يستدعي Service ويعيد قيمة، والإطار يكتب الاستجابة.

## التعريف التقني

توثيق Controllers: Controller مسؤول عن الطلبات الواردة وعن إرجاع الاستجابات. آلية Routing تختار الصنف، ولكل مسار فعل مختلف.

`@Controller()` شرط حتى يدخل الصنف الخريطة. تسجيله في `controllers` داخل Module هو ما يجعل Nest تنشئه. الملف على القرص وحده لا ينشر مسارًا.

الأمر `nest g controller shelves` ينشئ الملف ويضيف الصنف غالبًا إلى Module القريب. راجع المصفوفة بعد التوليد.

## Mental Model

```text
طلب HTTP
  -> خريطة المسارات
  -> دالة داخل Controller
  -> Service
  -> قيمة معادة
  -> Nest تكتب الاستجابة
```

اسم الدالة لا يحدد المسار. الربط يتم بالـ Decorator على الدالة، والاسم للمبرمج.

## أبسط مثال

```ts
import { Controller, Get } from '@nestjs/common';
import { ShelvesService } from './shelves.service.js';

@Controller('shelves')
export class ShelvesController {
  constructor(private readonly shelves: ShelvesService) {}

  @Get()
  findAll() {
    return this.shelves.findAll();
  }
}
```

## شرح المثال

- `@Controller('shelves')` بادئة المجموعة. تفصيلها في الدرس التالي.
- `@Get()` يربط `findAll` بطلب قراءة على تلك البادئة.
- `constructor` يطلب `ShelvesService`. Nest تمرر النسخة المسجّلة. كتابة الحقن في فصل Providers.
- إعادة قيمة الخدمة هي الأسلوب القياسي. Nest تسلسل المصفوفة أو الكائن JSON.

## ماذا يحدث داخليًا؟

عند الإقلاع تُقرأ الـ Metadata وتُسجَّل المعالجات. الطلب المطابق يستدعي الدالة. القيمة تذهب إلى طبقة الاستجابة في Nest. الاستثناء غير الممسوك يصل لاحقًا إلى Exception Filter، والترتيب الكامل في فصل Request Lifecycle.

النسخة الافتراضية واحدة لكل الطلبات. Node خيط واحد، فالمشاركة آمنة ما دام المعالج لا يخزن حالة طلب على `this`. حالة الطلب تبقى في وسائط الدالة. نطاق REQUEST يُشرح مع Providers.

## مثال عملي

```ts
import { Module } from '@nestjs/common';
import { ShelvesController } from './shelves.controller.js';
import { ShelvesService } from './shelves.service.js';

@Module({
  controllers: [ShelvesController],
  providers: [ShelvesService],
})
export class ShelvesModule {}
```

`GET /shelves` يصل إلى `findAll` فقط بعد أن تستورد شجرة التطبيق `ShelvesModule`. المعالج لا يعرف مكان حفظ الرف. `openNext` و `findAll` داخل الخدمة هما مكان القاعدة، والمساران في Controller يبقيان رفيعين.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| وضع الاستعلام داخل المعالج | القاعدة تتكرر عند كل مسار ويصعب اختبارها وحدها |
| نسيان `controllers: [ShelvesController]` | الملف موجود و `GET /shelves` لا يجد معالجًا |
| حفظ `lastVisitor` كحقل في الصنف | الطلب التالي يقرأ زائر الطلب السابق |
| Controller واحد لكل التطبيق | البادئات تضيع في صنف بلا حد ميزة |

## متى نستخدمه؟

لمورد HTTP واضح: رفوف، نسخ، إعارات. كل مجموعة مسارات مترابطة تأخذ Controller في مجلد ميزتها.

## متى لا نستخدمه؟

لحساب غرامة أو اختيار مخزن. ذلك Service. رفض الطلب قبل اختيار المعالج عمل Guard أو Middleware، وفصلهما لاحق. هنا المعالج يستقبل ثم يفوّض.

## مقارنة

| Express | Controller في Nest |
| --- | --- |
| `app.get` يسجل دالة حرة | Decorator على دالة داخل صنف مسجّل |
| المعالج يكتب `res.json` غالبًا | المعالج يعيد قيمة وNest تكتب الجسم |
| الدالة تملك القاعدة والرد | الدالة تملك شكل الطلب، والقاعدة في Service |

## تمارين

### تمرين سهل

ماذا تعيد دالة `findAll` في المثال؟

<details><summary>الحل</summary>

نتيجة `this.shelves.findAll()`. Nest تحوّل المصفوفة أو الكائن إلى JSON.

</details>

### تمرين متوسط

الصنف مكتوب، و `GET /shelves` لا يطابق معالجًا. أين النظر أولًا؟

<details><summary>الحل</summary>

مصفوفة `controllers` في Module الواصلة إلى الجذر. من غير تسجيل لا تُنشأ النسخة ولا تُبنى الخريطة.

</details>

### تمرين متقدم

لماذا حقل `lastVisitor` على Controller يخلط زائرين، بينما القيمة المعادة من الدالة تبقى لطلب واحد؟

<details><summary>الحل</summary>

النطاق الافتراضي نسخة واحدة. الحقل يعيش عليها فيراه الطلب التالي. القيمة المعادة محلية للاستدعاء، وتُكتب في استجابة ذلك الطلب.

</details>

## أسئلة مراجعة

- ما مسؤولية Controller كما في التوثيق؟
- لماذا لا يحدد اسم الدالة المسار؟
- أين تُكتب قاعدة العمل عندما يكون المعالج رفيعًا؟

## الخلاصة

Controller صنف مسجّل يستقبل الطلب، يفوّض Service، ويعيد قيمة. الخريطة من الـ Decorators، والنسخة الافتراضية مشتركة بين الطلبات.

## ماذا بعد؟

Decorator `@Controller`: البادئة وشكل وسيطها.

## المصادر الرسمية

- NestJS — Controllers: https://docs.nestjs.com/controllers
