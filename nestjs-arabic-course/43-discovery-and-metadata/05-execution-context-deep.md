# ExecutionContext بعمق

## الهدف من هذا الدرس

استخدام `ExecutionContext` لمعرفة نوع القناة والهدف، ثم التبديل إلى HTTP أو RPC أو WebSocket من غير افتراض أن كل طلب كائن Express.

## المشكلة التي يحلها

حارس منسوخ من مثال HTTP يستدعي `switchToHttp().getRequest()` داخل معالج رسائل فيصبح `request` فارغًا أو بلا `headers`. السياق يغلّف وسائط الاستدعاء ويعرف إن كان المصدر HTTP أو RPC أو WS. التوثيق يبني على ذلك حياد المنصة: المكتبة المشتركة تسأل `getType()` قبل أن تلمس شكل المنصة.

## الفكرة ببساطة

`ExecutionContext` يمتد مضيف الوسائط. `getType()` يعيد `'http'` أو `'rpc'` أو `'ws'`. `switchToHttp()` يعطي `getRequest` و `getResponse` و `getNext`. `switchToRpc()` يعطي `getData` و `getContext`. `switchToWs()` يعطي العميل والبيانات. `getHandler()` و `getClass()` هدف metadata الذي قرأه `Reflector`. لا تخزن السياق على Singleton بعد انتهاء الدالة.

## التعريف التقني

في حارس:

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class ChannelGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    if (context.getType() === 'http') {
      const request = context.switchToHttp().getRequest<{ user?: { id: string } }>();
      return Boolean(request.user);
    }
    if (context.getType() === 'rpc') {
      const data = context.switchToRpc().getData<{ userId?: string }>();
      return Boolean(data.userId);
    }
    return false;
  }
}
```

`getArgByIndex(0)` وسيط خام إن احتجت موقعًا بعينه. في HTTP الترتيب الشائع request ثم response ثم next، لكن التبديل إلى المضيف أوضح من الفهرس.

معالج الرسائل:

```ts
import { Controller } from '@nestjs/common';
import { Ctx, MessagePattern, Payload, type RmqContext } from '@nestjs/microservices';

@Controller()
export class BillingController {
  @MessagePattern('invoice.open')
  open(@Payload() data: { id: string }, @Ctx() ctx: RmqContext): { id: string } {
    void ctx;
    return data;
  }
}
```

الحارس على هذا المسار يرى النوع `rpc`. `@Payload()` داخل المعالج أوضح من إعادة التبديل إن كنت أصلًا في Controller الرسائل. الحارس المشترك هو من يحتاج `getType`.

WebSocket: `switchToWs().getClient()` و `getData()`. البوابة تبقى Singleton كما في فصل النطاقات، والسياق يصف الرسالة لا عمر البوابة.

`getHandler` و `getClass` تعمل في القنوات الثلاث، لذلك `@Public()` و `@Roles()` تُقرآن بالطريقة نفسها ثم يختلف استخراج الفاعل.

## Mental Model

```text
ExecutionContext
  getType: http | rpc | ws
  getHandler / getClass -> Reflector
  switchTo* -> شكل هذه القناة فقط
```

لا تمرر كائن Express إلى خدمة النطاق. استخرج `userId` نصًا.

## أبسط مثال

`ChannelGuard` أعلاه. سجّله حيث يصل HTTP و microservice معًا (تطبيق هجين). مسار بلا `user` في HTTP يُرفض. رسالة RPC بلا `userId` تُرفض. نوع غير معروف يُرفض صراحة بدل قراءة `headers` من `undefined`.

## شرح المثال

`switchToHttp` على سياق RPC لا يعطيك طلب متصفح. الشرط على `getType` يحمي الاستخراج. أعد `false` أو ارمِ استثناء يناسب القناة: `HttpException` للمتصفح و `RpcException` للرسائل، وذلك يُفصل في فصل الخدمات الصغيرة. الحارس الذي يرمي HTTP فقط يكسر العميل RPC.

## ماذا يحدث داخليًا؟

1. الإطار يبني السياق من وسائط الاستدعاء ومن معرّف القناة.
2. الحارس يستلمه قبل المعالج في مسار HTTP، وبشكل مماثل في القنوات التي تدعم الحراس.
3. `getHandler` يشير إلى الدالة التي تحمل metadata.
4. `switchToHttp().getRequest()` مرجع طلب هذه الدعوة. لا تحتفظ به على حقل خدمة `DEFAULT` بعد `return`.
5. في RPC `getData()` حمولة الرسالة لا كائن `Request`.
6. في WS العميل اتصال طويل. لا تعلّم البوابة `REQUEST`.

## مثال عملي

استخراج معرف واحد للدومين:

```ts
function actorId(context: ExecutionContext): string | undefined {
  switch (context.getType()) {
    case 'http':
      return context.switchToHttp().getRequest<{ user?: { id: string } }>().user?.id;
    case 'rpc':
      return context.switchToRpc().getData<{ userId?: string }>().userId;
    default:
      return undefined;
  }
}
```

الخدمة تستقبل `string`. الاختبار يستدعي الخدمة بلا سياق. هذا حد حياد المنصة على مقياس الدالة.

## أخطاء شائعة

- افتراض HTTP في حارس عام على تطبيق هجين.
- تخزين `request` على Singleton.
- `switchToHttp` لقراءة أدوار. الأدوار من `Reflector` على `getHandler`.
- معاملة عميل WS كنطاق طلب.
- رمي `BadRequestException` من مسار RPC وتوقع أن الوسيط يفهم جسم HTTP.

## متى نستخدمه؟

حارس أو معترض مشترك بين القنوات، أو عندما تحتاج الدالة والصنف مع شكل الاستدعاء.

## متى لا نستخدمه؟

داخل خدمة دومين. مرر قيمًا plain. داخل متحكم HTTP فقط، `@Req()` يكفي إن كانت الدالة أصلًا على المنصة.

## مقارنة

| `getType()` | المضيف | ما تستخرجه عادة |
| --- | --- | --- |
| `http` | `switchToHttp` | `user` من الطلب |
| `rpc` | `switchToRpc` | حقول الحمولة |
| `ws` | `switchToWs` | العميل وبيانات الرسالة |

## تمارين

### تمرين سهل

أي دالتين تغذيان `Reflector` من السياق؟

<details>
<summary>الحل</summary>

`getHandler()` و `getClass()`.

</details>

### تمرين متوسط

لماذا يفشل حارس يقرأ `request.headers` على `@MessagePattern`؟

<details>
<summary>الحل</summary>

لأن النوع `rpc` والحمولة في `getData()`، لا في طلب HTTP.

</details>

### تمرين متقدم

كيف تبقى `OrdersService` جاهلة بـ Express و RPC معًا؟

<details>
<summary>الحل</summary>

الحارس أو المتحكم يستخرج `userId: string` عبر `getType()` ويمرره وسيطًا. الخدمة لا تستقبل `ExecutionContext`.

</details>

## أسئلة مراجعة

- ماذا يعيد `getType()`؟
- لماذا لا يُخزن الطلب على خدمة افتراضية؟
- أي استثناء يناسب كل قناة بشكل عام؟

## الخلاصة

`ExecutionContext` يسمي القناة والهدف. بدّل إلى مضيف تلك القناة فقط، واقرأ metadata من الدالة والصنف، ومرر إلى الدومين قيمًا لا كائنات منصة.

## ماذا بعد؟

أمثلة تجمع `@Public()` مع مسح أعلام وقناة RPC.

## المصادر الرسمية

- NestJS — Execution context: https://docs.nestjs.com/fundamentals/execution-context
- NestJS — Custom decorators: https://docs.nestjs.com/custom-decorators
- NestJS — Discovery service: https://docs.nestjs.com/fundamentals/discovery-service
