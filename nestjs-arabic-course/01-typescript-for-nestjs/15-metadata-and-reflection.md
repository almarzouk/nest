# Metadata و Reflection

## الهدف من هذا الدرس

فهم كيف يبقى نوع `UsersService` مفيدًا بعد أن يمسح المترجم الأنواع، وكيف تجد Nest الوصف الذي لصقته Decorators.

## المشكلة التي يحلها

JavaScript الناتج لا يعرف أن معامل constructor كان `UsersService`. من غير معلومة إضافية، الحاوية ترى وسيطًا بلا نوع. كذلك `@Get(':id')` يحتاج مخزنًا يبقى على الدالة حتى الإقلاع.

Reflection هنا يعني: سؤال الهدف عن وصف لُصق عليه سابقًا، لا قراءة أفكار البرنامج.

## الفكرة ببساطة

`Reflect.defineMetadata(key, value, target)` يخزن قيمة على هدف بمفتاح. `Reflect.getMetadata` يقرأها. `emitDecoratorMetadata` يجعل المترجم يخزن أنواع التصميم تلقائيًا بجانب الزينة. مكتبة `reflect-metadata` تعرّف `Reflect.metadata` وعمليات القراءة، وهي من الحزم الدنيا التي يسميها توثيق Nest.

## التعريف التقني

Metadata بيانات مرتبطة بهدف (صنف، أو prototype دالة، أو معامل) بمفتاح. التصميم الذي يصدره TypeScript يستخدم مفاتيح مثل `design:paramtypes` لأنواع معاملات الدالة المزينة، و `design:type` لنوع الخاصية، و `design:returntype` لنوع الإرجاع.

الإصدار يحدث فقط إذا:

1. `emitDecoratorMetadata` مفعّل.
2. توجد Decorator واحدة على الأقل على الهدف (الصنف أو الدالة). بلا زينة لا تُصدر هذه المفاتيح للتصميم.
3. تُحمَّل `reflect-metadata` مرة عند إقلاع العملية قبل إنشاء التطبيق. قالب Nest يفعل ذلك في المدخل.

أنواع تُمسح أو تصير `Object` في هذه المفاتيح: الواجهات، والاتحادات، والـ generics الممحوة. لهذا الحقن يعتمد على أصناف ملموسة، والتحقق يعتمد على أصناف لا interfaces.

## Mental Model

```text
constructor(private readonly usersService: UsersService)

مع @Injectable على الصنف
        |
        v
المترجم يضيف design:paramtypes = [UsersService]
        |
        v
Nest عند الإقلاع: المعامل 0 مفتاحه UsersService
        |
        v
الحاوية تبحث عن مزوّد سُجّل بهذا الصنف
```

## أبسط مثال

```ts
import 'reflect-metadata';

const KEY = 'route:path';

function Path(path: string): ClassDecorator {
  return (target) => {
    Reflect.defineMetadata(KEY, path, target);
  };
}

@Path('/books')
class BooksController {}

const saved = Reflect.getMetadata(KEY, BooksController) as string;
```

## شرح المثال

- `import 'reflect-metadata'` يجهّز `Reflect.defineMetadata`. من غيره تفشل الاستدعاءات أو لا توجد.
- `KEY` مفتاح نتفق عليه. Nest تستخدم مفاتيحها داخل الحزم، لا مفتاحك هذا.
- `Path` مصنع يعيد Decorator صنف.
- داخل الزينة، `target` هو `BooksController`.
- `defineMetadata` يربط `'/books'` بالصنف تحت المفتاح.
- `getMetadata` يعيد `'/books'` لاحقًا من أي مكان يملك الصنف والمفتاح.
- `saved` نص المسار. لا يوجد خادم بعد. الوصف فقط.

## ماذا يحدث داخليًا؟

المكتبة تضع القيم في مخزن ضعيف مرتبط بالهدف حتى لا تبقى الكائنات حية بلا داعٍ فقط بسبب الوصف، حسب تصميم `reflect-metadata`. المهم لك: الوصف يعيش على الصنف بعد تحميل الملف، والإقلاع يمشيه.

لحقن الاعتماديات يصدر المترجم ما يشبه:

```js
__metadata('design:paramtypes', [UsersService]);
```

بجانب زينة الصنف. `UsersService` هنا مرجع للدالة البناء، لا اسم نصي فقط. لذلك الصنف يجب أن يكون معرفًا كقيمة. `import type` يمسح المرجع، فينكسر الحقن أو البناء. استورد الخدمات استيراد قيمة:

```ts
import { UsersService } from './users.service.js';
```

لا:

```ts
import type { UsersService } from './users.service.js';
```

`import type` صحيح للأنواع الممحوة التي لا تلمسها Nest وقت التشغيل، مثل Interface داخل ملف لا يُحقن.

حدود Metadata التصميم:

| المكتوب | ماذا يرى design:paramtypes غالبًا |
| --- | --- |
| صنف `UsersService` | الدالة `UsersService` |
| `string` و `number` و `boolean` | الدوال المقابلة في اللغة |
| Interface أو type alias لكائن | `Object` |
| اتحاد | `Object` أو ما تبقى بعد المحو |
| Generic ممحو | لا تفاصيل الحقول |

لذلك `@Body() dto: CreateBookDto` يعمل مع صنف `CreateBookDto`، ولا يعمل الفحص إن كان `dto` واجهة. الحقن بالمعامل `usersService: UsersService` يعمل لأن `UsersService` صنف.

Nest تخزن أيضًا Metadata خاصة بالمسار عبر زينتها، بمفاتيح داخل `@nestjs/common`. `Reflector` لاحقًا يقرأ مفاتيحًا تعلنها أنت مع `SetMetadata`. لا تستخدم مفاتيح داخلية غير موثّقة.

SWC و Vitest في مشاريع Nest 12 يحتاجان مسارًا يصدر Metadata التصميم أيضًا. قالب CLI يضبط ذلك. إن بنيت بأداة تتجاهل الزينة، الإقلاع يفشل لأن الأنواع غير موجودة. هذا سبب ذكر التوثيق لـ SWC مع إعداد الإطار لا أي مترجم عشوائي.

## مثال عملي

صنف خدمة بلا Decorator، ومتحكم يطلبها:

```ts
class UsersService {}

@Controller()
class UsersController {
  constructor(private readonly usersService: UsersService) {}
}
```

`@Controller` يزين الصنف، فيمكن إصدار `design:paramtypes` للمتحكم لأن الزينة على الصنف. المعامل يُرى `UsersService`. لكن الخدمة نفسها إن لم تُسجَّل في `providers` فالحاوية لا تنشئها حتى لو عرفت النوع. Metadata تجيب "ماذا تريد؟". التسجيل يجيب "هل أملك واحدة؟". السؤالان معًا في فصلي Modules و DI.

`@Injectable()` على الخدمة مطلوب في الممارسة التي يوثّقها Nest حتى تُعامل كمزوّد قابل للحقن وتُصدر بياناتها إن كان لها اعتماديات. ستضعه دائمًا على الخدمات في هذه الدورة.

## أخطاء شائعة

- نسيان `import 'reflect-metadata'` في مدخل بنيناه يدويًا. قالب `nest new` يتضمنه.
- `import type` لخدمة تُحقن.
- توقع Metadata لحقول Interface.
- إطفاء `emitDecoratorMetadata` ويبقى `experimentalDecorators`. الزينة قد تعمل والمسارات تُسجل إن خزنت الزينة مسارها بنفسها، بينما أنواع المعاملات تختفي فيفشل الحقن الصامت أو يرمي عند الإقلاع.
- قراءة Metadata عند كل طلب بحثًا عن مسار. المسارات تُبنى عند الإقلاع. القراءة المتكررة لكل طلب لها أماكن أخرى مثل `Reflector` داخل Guard، وهي رخيصة مقارنة بقاعدة البيانات، لكنها ليست بديلًا عن جدول المسارات.

## متى نستخدم Reflection مباشرة؟

عندما تكتب Decorator خاصًا وتخزن مفتاحًا موثّقًا، ثم تقرأه بـ `Reflector` الذي تغلفه Nest. الفصل 16 في المسار (Custom Decorators) والدرس المتقدم Discovery. اليوم يكفي أن تعرف المخزن موجود.

## متى لا نستخدمه؟

لتخزين حالة الطلب. Metadata على الصنف مشتركة بين كل الطلبات لأنها على القالب لا على الطلب. حالة الطلب تذهب مع كائن الطلب أو مع قيمة ترجعها الدالة.

## مقارنة

| الآلية | ماذا تبقي بعد الترجمة |
| --- | --- |
| نوع عادي `: Book` | لا شيء |
| Interface | لا شيء |
| Decorator كتبتها | استدعاء دالة التخزين |
| `emitDecoratorMetadata` | مفاتيح `design:*` على الهدف المزين |
| تسجيل `providers` | قرار الحاوية عند الإقلاع |

## تمارين

### تمرين سهل

ماذا تعيد `getMetadata` في المثال إذا كان المفتاح `KEY` والقيمة `'/books'`؟

<details>
<summary>الحل</summary>

النص `'/books'`.

</details>

### تمرين متوسط

لماذا وجود `@Controller()` على الصنف يؤثر على قدرة المترجم على إصدار أنواع معاملات الـ constructor؟

<details>
<summary>الحل</summary>

إصدار `design:paramtypes` مرتبط بوجود Decorator على الهدف. الزينة سبب لإصدار التصميم، لا مجرد تزيين شكلي. بلا أي زينة على الصنف قد لا تُصدر أنواع المعاملات.

</details>

### تمرين متقدم

معامل `config: AppConfig` و `AppConfig` واجهة. ماذا تتوقع أن تجد الحاوية في `design:paramtypes`، وهل يكفي ذلك لحقن صحيح؟

<details>
<summary>الحل</summary>

غالبًا `Object` لا الواجهة، لأن الواجهة مُحيت. كل الواجهات تتشابه في هذه الخانة، فلا مفتاح فريد للحقن. الحل: صنف، أو رمز حقن `@Inject(TOKEN)` مع مزوّد مسجّل بذلك الرمز. الرموز درس Providers.

</details>

## أسئلة مراجعة

- ما دور حزمة `reflect-metadata`؟
- ما الشرطان الظاهران لإصدار `design:paramtypes`؟
- لماذا `import type` يكسر حقن خدمة؟
- أين تُخزَّن Metadata الصنف: على الطلب أم على القالب؟

## الخلاصة

Reflection مخزن وصف على الأصناف والدوال. Decorators تكتب فيه، و `emitDecoratorMetadata` يكتب أنواع التصميم للأصناف الملموسة، و Nest تقرأ ذلك عند الإقلاع لتربط المسارات والحقن. الواجهات لا تبقى. التسجيل في الوحدة ما زال مطلوبًا.

## ماذا بعد؟

ملف الأمثلة يجمع السطر الكامل كما سيبدو قبل إنشاء المشروع، ثم التمارين، ثم تثبيت Nest حيث ترى `tsconfig` الحقيقي.

## المصادر الرسمية

- TypeScript — `emitDecoratorMetadata`: https://www.typescriptlang.org/tsconfig/emitDecoratorMetadata.html
- حزمة `reflect-metadata`: https://github.com/rbuckton/reflect-metadata
- NestJS — Custom decorators: https://docs.nestjs.com/custom-decorators
- NestJS — Providers: https://docs.nestjs.com/providers
- NestJS — Validation (لا metadata للواجهات): https://docs.nestjs.com/application/validation
