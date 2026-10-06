# Headers

## الهدف من هذا الدرس

قراءة رؤوس الطلب الواردة بـ `@Headers`: رأس واحد أو المجموعة كلها.

## المشكلة التي يحلها

معرف التتبع ومفتاح المستودع يسافران في الرؤوس لأنهما بيانات عن الطلب، لا عن مورد الكتب في المسار ولا حقولًا في جسم الإعارة. قراءتها من `req.headers` داخل كل دالة تكرر الفهرس نفسه. `@Headers` يسمي الرأس عند الوسيط.

## الفكرة ببساطة

`@Headers('x-warehouse')` يقرأ ذلك الرأس. `@Headers()` يقرأ الكائن. Node يخزن أسماء الرؤوس بأحرف صغيرة، فتبحث عن `x-warehouse` حتى لو أرسل العميل `X-Warehouse`.

## التعريف التقني

`@Headers(name)` يأخذ `req.headers[name]`. `@Headers()` يأخذ `req.headers`. القيمة `string` أو `string[]` عندما يتكرر الرأس. افحص النوع قبل استدعاء دوال النص.

الرأس `authorization` يصل كما كتبه العميل، غالبًا `Bearer ...`. تحليل الهوية فصل Authentication. هنا نمرر النص إلى Service إن كانت العملية تحتاج إليه.

`@Header()` من غير s يكتب رأس استجابة ثابتًا على المعالج. هو أداة إخراج في الأسلوب القياسي، وموضعه الطبيعي مع إرجاع القيمة. هذا الدرس يقرأ المدخل.

## Mental Model

```text
GET /warehouses/stock
X-Warehouse: jeddah
X-Request-Id: 8f2

@Headers('x-warehouse') -> "jeddah"
@Headers('x-request-id') -> "8f2"
```

## أبسط مثال

```ts
import { Controller, Get, Headers } from '@nestjs/common';
import { StockService } from './stock.service.js';

@Controller('warehouses')
export class StockController {
  constructor(private readonly stock: StockService) {}

  @Get('stock')
  stockAt(@Headers('x-warehouse') warehouse?: string) {
    return this.stock.list(warehouse);
  }
}
```

## شرح المثال

- الاسم `x-warehouse` بالأحرف الصغيرة يطابق ما يخزنه Node.
- الغياب يعطي `undefined`. الخدمة ترفض الطلب أو تختار مستودعًا افتراضيًا بقاعدة ظاهرة.
- المسار `GET /warehouses/stock` ثابت. المستودع معلومة نقل لا جزءًا من مجموعة الرفوف في URL.
- إعادة قائمة الخدمة تُرسل JSON برمز 200.

## ماذا يحدث داخليًا؟

المحوّل يملأ `req.headers` قبل المعالج. Nest تقرأ المفتاح المطلوب. البحث حساس لحالة الأحرف في كائن JavaScript، لذلك `X-Warehouse` كمفتاح في `@Headers('X-Warehouse')` لا يجد القيمة بعد تصغير Node للاسم.

بعض الرؤوس مصفوفة. `accept` قد يبقى نصًا واحدًا فيه فواصل، بينما رأسًا مكررًا قد يصل `string[]`. مرّر إلى الخدمة الشكل الذي وثّقته، أو خذ أول عنصر بعد أن تتأكد من النوع.

## مثال عملي

```ts
import { Controller, Get, Headers } from '@nestjs/common';
import { StockService } from './stock.service.js';

@Controller('warehouses')
export class StockTraceController {
  constructor(private readonly stock: StockService) {}

  @Get('movements')
  movements(
    @Headers('x-warehouse') warehouse: string,
    @Headers('x-request-id') requestId: string | string[],
  ) {
    const trace = Array.isArray(requestId) ? requestId[0] : requestId;
    return this.stock.movements(warehouse, trace);
  }
}
```

`movements` تحتاج المستودع ورقم التتبع معًا. تحويل المصفوفة إلى نص يتم في المعالج لأنه تفصيل بروتوكول، ثم تتسلم الخدمة نصين.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `@Headers('X-Request-Id')` بالحالة الأصلية | المفتاح غير موجود بعد تصغير Node للاسم |
| معاملة كل رأس كـ `string` | رأس مكرر يرمي عند استدعاء `.slice` على مصفوفة |
| تحليل `authorization` إلى مستخدم داخل Controller | قاعدة الهوية تتوزع على المسارات قبل فصلها |
| الخلط مع `@Header('Cache-Control', 'no-store')` | الأول يقرأ طلبًا، والثاني يثبت رأس استجابة |

## متى نستخدمه؟

عندما تكون المعلومة عن النقل أو السياق: مستودع، معرف طلب، لغة `accept-language`، مفتاح idempotency. اقرأ المفتاح الذي تحتاجه فقط.

## متى لا نستخدمه؟

لتمرير ISBN أو رقم الصفحة. تلك `@Param` و `@Query` و `@Body`. ولكتابة `Cache-Control` على الاستجابة. تلك `@Header` مع إرجاع القيمة، أو `@Res` في درس Response Object.

## مقارنة

| Decorator | الاتجاه |
| --- | --- |
| `@Headers('x-warehouse')` | يقرأ رأس الطلب |
| `@Headers()` | كائن رؤوس الطلب |
| `@Header('Cache-Control', 'no-store')` | يثبت رأس استجابة على المعالج |

## تمارين

### تمرين سهل

أي سلسلة تمررها إلى `@Headers` لقراءة `X-Warehouse`؟

<details><summary>الحل</summary>

`'x-warehouse'`.

</details>

### تمرين متوسط

`x-request-id` وصل مصفوفة من قيمتين. ماذا تفعل قبل تمريره إلى `movements`؟

<details><summary>الحل</summary>

تفحص `Array.isArray`. إن صدقت خذ عنصرًا موثّقًا، غالبًا الأول، ومرّر نصًا. استدعاء دوال النص على المصفوفة يفشل.

</details>

### تمرين متقدم

لماذا يبقى `authorization` نصًا خامًا في هذا الفصل؟

<details><summary>الحل</summary>

الرأس مدخل بروتوكول. استخراج المستخدم والصلاحية قاعدة لاحقة في Authentication و Guards. Controller هنا يقرأ الرأس أو يمرره، ولا يبني جلسة من النص.

</details>

## أسئلة مراجعة

- ماذا يعيد `@Headers()` بلا اسم؟
- لماذا تُكتب أسماء الرؤوس صغيرة في الـ Decorator؟
- أي Decorator يثبت رأسًا على الاستجابة؟

## الخلاصة

`@Headers` يقرأ رؤوس الطلب بعد أن يصغّر Node أسماءها. القيمة قد تكون نصًا أو مصفوفة، ورأس الاستجابة الثابت يُكتب بـ `@Header`.

## ماذا بعد؟

رموز الحالة والتحويل: Status Codes و Redirects.

## المصادر الرسمية

- NestJS — Controllers (Request object): https://docs.nestjs.com/controllers#request-object
