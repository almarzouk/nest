# CanActivate

## الهدف من هذا الدرس

تنفيذ `canActivate` وفهم قيم الرجوع: `true` و `false` والاستثناء والوعد والمراقب.

## المشكلة التي يحلها

بوابة ترجع `true` دائمًا لا تحمي شباك التذاكر. الفريق يحتاج عقد الإجابة: متى يكمل الطلب، ومتى يصبح `403`، ومتى تختار أنت رمزًا آخر.

## الفكرة ببساطة

`canActivate` ترجع منطقيًا أو `Promise` أو `Observable` من تلك القيمة. `true` يكمل. `false` يجعل الإطار يرمي `ForbiddenException`. لرمز غير `403` ارمِ الاستثناء بنفسك، مثل `UnauthorizedException`.

## التعريف التقني

```ts
canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean>;
```

`ExecutionContext` درس التالي. هنا نستخدمه بما يكفي لقراءة الطلب.

عند `false` الرد الذي تذكره صفحة الحراس:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

أي استثناء ترميه الحارس تمضي به طبقة الاستثناءات: الفلتر العام وفلاتر السياق الحالي. المسار قد حُدد، بخلاف استثناء Middleware.

لا تلتقط `ForbiddenException` داخل الحارس وتعيد `false` من غير قصد مزدوج. إمّا ترجع `false` وتقبل رسالة الإطار، وإمّا ترمي الاستثناء الذي تريد ظهوره.

الدالة قد تكون `async`. الوعد المرفوض يذهب إلى طبقة الاستثناءات. الوعد الذي يحل إلى `false` يعادل الإرجاع المتزامن لـ `false`.

تمرير الصنف في `@UseGuards(BoxPassGuard)` يتيح الحقن. `new BoxPassGuard()` لا يحقن.

## Mental Model

```text
true                  → Interceptor ثم Pipe ثم الدالة
false                 → 403 ForbiddenException
throw Unauthorized    → 401 بالشكل الذي رميته
Promise / Observable  → الإطار ينتظر ثم يطبّق القاعدة نفسها
```

## أبسط مثال

```ts
import {
  type CanActivate,
  type ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';

@Injectable()
export class BoxPassGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<{ headers: Record<string, string> }>();
    const pass = request.headers['x-box-pass'];
    if (!pass) {
      throw new UnauthorizedException();
    }
    return pass === 'desk';
  }
}
```

غياب الترويسة `401`. وجودها بقيمة غير `desk` يرجع `false` فيصبح `403`. القيمة `desk` ترجع `true`. هذا تمرين آلية على ترويسة واضحة، لا مصادقة إنتاج. الرمز الحقيقي يُستبدل في فصل المصادقة.

## شرح المثال

- الرمي عند الغياب يفرّق «لم يقدّم نفسه» عن «قدّم قيمة لا تفتح هذا الشباك».
- `return pass === 'desk'` قد يكون `false` من غير `throw`. الإطار يكتب `403`.
- لا تفحص `seat` هنا.
- الترويسة مثال تعليمي ثابت حتى يبقى الدرس عن قيمة الرجوع.

## ماذا يحدث داخليًا؟

Nest ينتظر الناتج إن كان وعدًا أو `Observable`. بعد الحصول على منطقي: `true` يستدعي الحارس التالي أو يكمل الدورة. `false` يرمي `ForbiddenException` داخل طبقة الاستثناءات. الأنابيب لا تعمل في الحالتين المرفوضتين.

`try/catch` في المتحكم لا يرى هذا الرمي لأنه حدث قبل الدالة. فلتر على المسار يراه، ثم فلتر المتحكم، ثم العام، وأول من يمسك يوقف الباقي.

حارس ثانٍ على المسار نفسه لا يُستدعى بعد `false` أو بعد الرمي. لا توجد سلسلة «إن رفض الأول جرّب الثاني» إلا إذا رتّبت الشرط داخل حارس واحد.

## مثال عملي

`POST /tickets` عليه `BoxPassGuard`. طلب بلا ترويسة من أداة مراقبة يرجع `401` ولا يفحص جسم المقعد. طلب بترويسة `guest` يرجع `403`. طلب `desk` يصل إلى `ValidationPipe`. هكذا تختبر ثلاث نتائج من غير قاعدة بيانات: اكتمال، ممنوع، غير معرّف.

## أخطاء شائعة

- إرجاع `undefined`. القيمة ليست `true`، والسلوك ليس عقدًا واضحًا. أرجع منطقيًا صريحًا.
- رمي `Error` عادي فيصبح `500`. استخدم `HttpException` أو ارجع `false`.
- إرجاع `false` حين تقصد `401`. `false` يعني `403` فقط.
- ابتلاع الاستثناء داخل `canActivate` وإرجاع `true` عند الفشل. البوابة تنفتح عند العطل.

## متى نستخدمه؟

في كل حارس: جواب صريح أو استثناء HTTP مقصود. `false` حين تناسب رسالة المنع الافتراضية. الرمي حين يكون الرمز `401` أو جسم خطأ عندكم.

## متى لا نستخدمه؟

لا ترجع كائن مستخدم من `canActivate`. الدالة قرار لا تحويل. إلصاق `request.user` ممكن قبل `return true`، وفصل المصادقة سيضع ذلك في مكانه. لا ترجع بيانات الجسم المحوّلة.

## مقارنة

| الناتج | ماذا يرى العميل |
| --- | --- |
| `true` | يكمل إلى بقية الدورة |
| `false` | `403` و `Forbidden resource` |
| `throw new UnauthorizedException()` | `401` |
| وعد مرفوض بـ `HttpException` | حالة ذلك الاستثناء |

## تمارين

### تمرين سهل

ماذا يرمي الإطار حين ترجع `canActivate` القيمة `false`؟

<details>
<summary>الحل</summary>

`ForbiddenException`، والرد `403` برسالة `Forbidden resource`.

</details>

### تمرين متوسط

تريد `401` عند غياب الترويسة لا `403`. ماذا تفعل بدل `return false`؟

<details>
<summary>الحل</summary>

`throw new UnauthorizedException()` قبل أي `return false`.

</details>

### تمرين متقدم

`canActivate` أصبحت `async` وتقرأ الترويسة من خدمة. الوعد يُرفض بـ `UnauthorizedException`. هل يعمل `try/catch` داخل `reserve`؟

<details>
<summary>الحل</summary>

لا. الرفض يحدث قبل استدعاء الدالة. طبقة الاستثناءات ترى الخطأ، ومنها فلتر المسار إن وُجد.

</details>

## أسئلة مراجعة

- ما الأنواع التي يجوز أن ترجعها `canActivate`؟
- ما رمز `false`؟
- كيف تحصل على `401`؟
- هل يُستدعى الأنبوب بعد الرفض؟

## الخلاصة

`canActivate` قرار: `true` يكمل، و `false` يمنع بـ `403`، والاستثناء الذي ترميه يحدد رمزًا آخر. الانتظار على الوعد لا يغيّر هذه القواعد. الرسالة الافتراضية للمنعون ليست مكانًا تخترع فيه `500`.

## ماذا بعد؟

الدرس التالي `ExecutionContext`: كيف تصل إلى الطلب والدالة والصنف من الوسيط نفسه.

## المصادر الرسمية

- [Guards](https://docs.nestjs.com/guards)
- [Execution context](https://docs.nestjs.com/fundamentals/execution-context)
