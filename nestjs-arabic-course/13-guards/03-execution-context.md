# ExecutionContext

## الهدف من هذا الدرس

قراءة سياق التنفيذ داخل الحارس: الطلب، وصنف المتحكم، ودالة المسار.

## المشكلة التي يحلها

حارس يرى `req` فقط يستطيع فحص ترويسة، ولا يستطيع معرفة أن الدالة الحالية هي `reserve` لا `list`. الصفحة تفرّق لهذا السبب بين Middleware الأعمى عن المعالج والحارس الذي يعرف ماذا سيُنفَّذ.

## الفكرة ببساطة

`ExecutionContext` يمتد من `ArgumentsHost`. منه `switchToHttp().getRequest()` للطلب، و `getHandler()` للدالة، و `getClass()` لصنف المتحكم. هذه المقابض هي ما سيقرأه `Reflector` لاحقًا على الزينة.

## التعريف التقني

`ArgumentsHost` يبدّل نوع النقل: `switchToHttp()` و `switchToRpc()` و `switchToWs()`. في تطبيق HTTP استخدم `switchToHttp()` ثم `getRequest()` و `getResponse()`. لا تفترض أن كائن HTTP هو نفسه كائن RPC.

`ExecutionContext` يضيف ما يخص التنفيذ الحالي:

- `getHandler()` مرجع الدالة التي ستُستدعى.
- `getClass()` صنف المتحكم.
- `getType()` نوع السياق مثل `'http'`.

الحارس العام الذي يعمل على HTTP و على ناقل آخر يفحص `getType()` قبل `switchToHttp()`. هذا الفصل يكتب أمثلة HTTP. الفصول اللاحقة للنقلات الأخرى.

لا تخزّن `request` على الحارس كحقل في الصنف. الحارس singleton على الطلبات كلها. اقرأ الطلب من الوسيط داخل `canActivate` فقط.

`getHandler()` لا يستدعي الدالة. هو مرجع للقراءة، وللبيانات الوصفية التي ستُلصَق في فصل الزينة.

## Mental Model

```text
ExecutionContext
  getClass()     TicketsController
  getHandler()   reserve
  switchToHttp() request / response
```

Middleware لا يملك هذا الكائن. لذلك قرار يعتمد على اسم الدالة يعيش هنا.

## أبسط مثال

```ts
import { type CanActivate, type ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class DeskRouteGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    if (context.getType() !== 'http') {
      return true;
    }
    const handler = context.getHandler();
    const controller = context.getClass();
    const request = context.switchToHttp().getRequest<{ method: string; url: string }>();
    void handler;
    void controller;
    return request.method === 'GET' || request.url.startsWith('/tickets');
  }
}
```

المثال يقرأ المقابض الثلاثة حتى ترى من أين تأتي. الشرط التعليمي يسمح بـ `GET` أو بمسار يبدأ بـ `/tickets`. استبدله بقاعدة أوضح في درس التفويض. لا تبنِ صلاحيات من `url` نصًا حين تتوفر بيانات الدالة لاحقًا.

## شرح المثال

- `getType()` يحمي حارسًا قد يُربَط في تطبيق فيه أكثر من نقل.
- `getHandler` و `getClass` مرجعان. طباعتهما في التطوير ترياك اسم الدالة واسم الصنف.
- `switchToHttp().getRequest()` طلب المنصة. على Express فيه `headers` و `method`.
- `void` في المثال حتى يبقى التركيز على الحصول على المرجع لا على استخدامه في قاعدة ناقصة.

## ماذا يحدث داخليًا؟

عند اختيار المسار يبني Nest سياقًا يشير إلى الدالة والصنف ووسائط الاستدعاء. الحارس يستلمه قبل Interceptor. قراءة `getHandler()` رخيصة. ما يُبنى فوقها لاحقًا هو `Reflector.get` على تلك الدالة ثم على الصنف.

`switchToHttp().getResponse()` متاح لكن كتابة الرد يدويًا من الحارس تتجاوز طبقة الاستثناءات. ارجع `false` أو ارمِ `HttpException` ودع الإطار يكتب الجسم.

السياق ليس `req` الذي رآه Middleware فقط. هو وصف للخطوة التالية في الدورة. لذلك تستطيع بوابة واحدة أن تسمح بـ `list` وتمنع `reserve` حين تُلصَق بيانات مختلفة على الدالتين. اللصق نفسه درس `@Roles` و `SetMetadata`.

## مثال عملي

أثناء التطوير اطبع `controller.name` و `handler.name` لطلب `POST /tickets`. ترى `TicketsController` و `reserve`. طلب `GET /health` على متحكم آخر يرى اسمًا مختلفًا إن كان الحارس عامًا. هذا التشخيص يفسر لماذا بوابة واحدة تستطيع أن تتفرع حسب الدالة، ولماذا Middleware الذي طُبع فيه المسار فقط لا يرى الاسم نفسه بالطريقة نفسها.

## أخطاء شائعة

- حفظ `request` على `this` فيستخدم طلب سابق في طلب لاحق.
- `switchToRpc()` في تطبيق HTTP فقط ثم قراءة حقول غير موجودة.
- استدعاء `getHandler()` كأنها الدالة: `getHandler()()`. التنفيذ ما زال مسؤولية الدورة بعد كل البوابات.
- مقارنة `url` نصًا لكل صلاحية. المسار يتغير مع البادئة. بيانات الدالة أثبت حين تُكتب.

## متى نستخدمه؟

في كل `canActivate` تحتاج الطلب أو هوية الدالة أو الصنف. وفي أي حارس قد يعمل خارج HTTP فتفحص `getType()` أولًا.

## متى لا نستخدمه؟

لا تقرأ السياق في Middleware. الواجهة غير موجودة هناك. لا تستخدم `getResponse()` لكتابة `403` يدويًا ما دام `false` يؤدي ذلك.

## مقارنة

| المقبض | يعيد | لا يفعل |
| --- | --- | --- |
| `getHandler()` | دالة المسار | لا يستدعيها |
| `getClass()` | صنف المتحكم | لا يبني نسخة جديدة |
| `switchToHttp()` | مضيف HTTP | لا يصلح وحده لـ RPC |
| `getRequest()` | طلب هذه الدورة | لا يُخزَّن على singleton |

## تمارين

### تمرين سهل

أي دالة تعيد مرجع معالج المسار من غير أن تستدعيه؟

<details>
<summary>الحل</summary>

`context.getHandler()`.

</details>

### تمرين متوسط

لماذا يفسد حفظ `request` في حقل على `BoxPassGuard`؟

<details>
<summary>الحل</summary>

الحارس يُشارَك بين الطلبات. الحقل يخلط طلبًا بطلب. اقرأ الطلب من `ExecutionContext` داخل الاستدعاء.

</details>

### تمرين متقدم

حارس عام موصول أيضًا بناقل رسائل. ماذا تفحص قبل `switchToHttp()`؟

<details>
<summary>الحل</summary>

`context.getType()`. إن لم تكن `'http'` لا تقرأ طلب HTTP. لكل نقل مبدّله، وفصول النقل لاحقًا.

</details>

## أسئلة مراجعة

- ماذا يضيف `ExecutionContext` على `ArgumentsHost`؟
- كيف تصل إلى طلب HTTP؟
- لماذا لا يخزَّن الطلب على الصنف؟
- أي فرق هذا عن Middleware؟

## الخلاصة

السياق يعطيك الدالة والصنف والطلب للخطوة الحالية. اقرأه داخل `canActivate` ولا تحتفظ به. هذه المقابض هي ما يجعل البوابة تعرف أي مسار ستحمي، وهي ما ستُبنى عليه قراءة الزينة لاحقًا.

## ماذا بعد؟

الدرس التالي يفرّق المصادقة عن التفويض مفاهيميًا، ويبقي بناء الرمز الحقيقي خارج هذا الفصل.

## المصادر الرسمية

- [Guards](https://docs.nestjs.com/guards)
- [Execution context](https://docs.nestjs.com/fundamentals/execution-context)
