# Platform Agnosticism

## الهدف من هذا الدرس

إبقاء معالج الرسالة والخدمة جاهلين بتفاصيل Express أو ناقل بعينه، عبر أنواع Nest و `ExecutionContext` لا عبر كائنات المنصة.

## المشكلة التي يحلها

خدمة تستورد `Request` من Express ثم تُنقل إلى `createMicroservice` فينكسر المعالج لأن لا طلب HTTP. التوثيق يبني Nest فوق محول منصة: HTTP قد يكون Express أو Fastify، والخدمة الصغيرة ناقلًا. الكود المشترك يتعامل مع تجريد الإطار. هذا حياد المنصة.

## الفكرة ببساطة

المتحكم يأخذ الحمولة كمعامل الدالة أو `@Payload()`. الحارس الذي يعيش على HTTP و RPC يسأل `getType()` قبل `switchToHttp` أو `switchToRpc`. الخدمة تستقبل `orderId: string`. لا `ClientKafka` داخل الدومين. تبديل TCP إلى NATS يمس `main.ts` وخيارات العميل لا قاعدة الإلغاء.

## التعريف التقني

صفحة Platform agnosticism تشرح أن المكونات نفسها تعمل عبر HTTP و WebSockets والخدمات الصغيرة قدر الإمكان، لأن التفاصيل خلف محول. عمليًا:

- لا تستورد `express` في ملف خدمة.
- DTO للحمولة كما لجسم HTTP.
- `RpcException` عندما يجب أن يفهم العميل البعيد الخطأ، لا `BadRequestException` وحدها على مسار رسالة.
- `ExecutionContext.switchToRpc().getData()` في حارس مشترك.
- اختبار المعالج بنداء `execute` أو الدالة مباشرة بلا خادم.

```ts
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';

@Controller()
export class InvoiceController {
  constructor(private readonly invoices: InvoiceService) {}

  @MessagePattern({ cmd: 'invoice.issue' })
  issue(@Payload() data: { orderId: string }) {
    return this.invoices.issue(data.orderId);
  }
}
```

`InvoiceService.issue` بلا نقل. نفس الدالة قد يناديها متحكم HTTP أثناء هجرة تدريجية إن مررت `orderId` فقط. بعد الهجرة يبقى المسار الرسائلي.

تغيير `Transport.TCP` إلى `Transport.NATS` في خيارات التطبيق والعميل لا يغيّر `issue` إن بقي النمط والحمولة. الاختلافات الحقيقية في الضمان والمهلة تُعالج في الخيارات لا في `if (kafka)` داخل الخدمة.

## Mental Model

```text
Transport في main والعميل
@Payload في المتحكم
InvoiceService بلا منصة
حارس: getType ثم switchTo*
```

## أبسط مثال

المقطع أعلاه. وحدة الاختبار تبني `InvoiceService` مع مستودع ذاكرة وتنادي `issue('o1')`. لا تستورد `@nestjs/microservices` في ملف الاختبار هذا.

## شرح المثال

`@Payload()` أوضح من أخذ البيانات كأول معامل عندما تضيف `@Ctx()` ثانيًا لسياق الناقل مثل إقرار Rabbit. الخدمة لا ترى السياق. المتحكم يقر الرسالة بعد نجاح `issue` إن كان الناقل يتطلب ذلك في المحول، وتبقى السياسة في الخدمة.

## ماذا يحدث داخليًا؟

1. المحول يفك التسلسل إلى حمولة.
2. الإطار يطابق النمط ويمرر الحمولة إلى الدالة.
3. Pipes يمكن أن تتحقق من الشكل على المعاملات المعلّمة.
4. الاستثناء إن بقي HTTP-only لا يُسلسل كما يتوقع عميل الناقل.
5. تبديل المحول يبدّل الخطوة 1 و 4 لا جسم `issue`.
6. هجين HTTP+ناقل يشترك في الخدمة نفسها خلف مدخلين رقيقين.

## مثال عملي

حارس هوية: على HTTP يقرأ `request.user.id`، وعلى RPC يقرأ `userId` من الحمولة التي وضعها البوابة بعد التحقق. الخدمة ترى `userId` في وسيط `issue(orderId, userId)`. لا تكرر فحص JWT داخل الفوترة إن البوابة تحققته. الوثوق بالحمولة الداخلية قرار شبكة: الناقل ليس عامًا على الإنترنت بلا حدود.

## أخطاء شائعة

- `import { Request } from 'express'` في الخدمة.
- فرع `if (process.env.TRANSPORT)` داخل الدومين.
- رمي `HttpException` فقط على `@MessagePattern`.
- تمرير كائن المقبس إلى المستودع.
- اختبار يحتاج وسيط Kafka لقاعدة الإلغاء.

## متى نستخدمه؟

في كل معالج رسالة وخدمة مشتركة بين مدخلين.

## متى لا نستخدمه؟

خيارات الاتصال (منفذ، وسطاء، مجموعة مستهلك) مكانها تكوين الناقل في `main` والعميل، وهي منصة عن قصد.

## مقارنة

| الطبقة | تعرف الناقل؟ |
| --- | --- |
| `main.ts` والعميل | نعم، `Transport` والخيارات |
| المتحكم | النمط والحمولة و `@Ctx` عند الحاجة |
| الخدمة والمستودع | معرفات وقيم فقط |
| الحارس المشترك | `getType()` ثم المضيف المناسب |

## تمارين

### تمرين سهل

ماذا تستقبل `InvoiceService` من المثال؟

<details>
<summary>الحل</summary>

`orderId` نصًا، لا طلب Express ولا سياق الناقل.

</details>

### تمرين متوسط

ماذا يتغير عند الانتقال من TCP إلى NATS إن التزمت بالحياد؟

<details>
<summary>الحل</summary>

خيارات النقل في المستمع والعميل. النمط والحمولة و `issue` يبقون.

</details>

### تمرين متقدم

لماذا `BadRequestException` وحدها على مسار رسالة قد لا تكفي؟

<details>
<summary>الحل</summary>

العميل البعيد يتوقع خطأ نقل مثل `RpcException` يُسلسل عبر الناقل. استثناء HTTP مفهومه جسم متصفح.

</details>

## أسئلة مراجعة

- أين يُستورد `Transport`؟
- ماذا يفعل `@Payload`؟
- لماذا اختبار السياسة بلا ناقل؟

## الخلاصة

حياد المنصة يعني سياسة بلا Express وبلا عميل ناقل. المدخل يفك الحمولة، والخيارات تعيش في الإقلاع، و `RpcException` عندما يعبر الخطأ العملية.

## ماذا بعد؟

`@MessagePattern` و `@EventPattern` بالتفصيل على المتحكم.

## المصادر الرسمية

- NestJS — Platform agnosticism: https://docs.nestjs.com/fundamentals/platform-agnosticism
- NestJS — Microservices: https://docs.nestjs.com/microservices/basics
