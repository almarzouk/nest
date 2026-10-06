# gRPC و MQTT ونواقل أخرى

## الهدف من هذا الدرس

معرفة متى تختار gRPC أو MQTT بدل أنماط Nest النصية، وكيف تعبر الأخطاء بـ `RpcException` على أي ناقل.

## المشكلة التي يحلها

ليس كل تكامل `@MessagePattern` على TCP. عقد صارم مع فرق تستعمل protobuf يناسبه gRPC. أجهزة ومواضيع خفيفة تناسب MQTT. الخطأ الذي يُرمى كـ `HttpException` لا يُفهم على هذه النواقل. `RpcException` الشكل الذي يسلسله الإطار نحو `ClientProxy`.

## الفكرة ببساطة

`Transport.GRPC` يحتاج ملف `.proto` وحزمة `@grpc/grpc-js` و `@grpc/proto-loader`. الدالة `@GrpcMethod` أو `@MessagePattern` حسب صفحة gRPC في التوثيق. `Transport.MQTT` يحتاج حزمة `mqtt` وعنوان `mqtt://broker:1883`. بقية النواقل في الوثائق (Kafka و NATS و Redis و Rabbit و TCP) مرت في دروسها. اختر بالضمان لا بالاسم الشائع.

`new RpcException('order missing')` أو `new RpcException({ message: 'order missing' })` من المعالج. العميل يرى الخطأ على Observable الـ `send`.

## التعريف التقني

gRPC شكل عقد:

```ts
{
  transport: Transport.GRPC,
  options: {
    package: 'billing',
    protoPath: join(process.cwd(), 'proto/billing.proto'),
    url: process.env.GRPC_URL ?? '0.0.0.0:5000',
  },
}
```

العميل يشير إلى `billing:5000` في Compose لا إلى `localhost` إن عبر الحاويات. الأسلوب في الـ proto هو العقد، أقوى من كائن `{ cmd }` حر. مناسب لحدود فرق تتفق على ملف. ثقيل للتعلم إن كان الطرفان Nest فقط وما زال العقد يتغير كل يوم.

MQTT:

```ts
{
  transport: Transport.MQTT,
  options: { url: process.env.MQTT_URL ?? 'mqtt://127.0.0.1:1883' },
}
```

المواضيع أحداث أجهزة أو إشارات خفيفة. لا تجعله قاعدة فواتير.

```ts
import { RpcException } from '@nestjs/microservices';

if (order === null) {
  throw new RpcException({ message: 'order missing' });
}
```

في التطبيق الهجين ارمِ `HttpException` من مسار HTTP و `RpcException` من مسار الرسالة، أو ترجم في فلتر حسب `getType()`.

## Mental Model

```text
عقد protobuf ثابت     -> gRPC
موضوع جهاز خفيف       -> MQTT
خطأ يعبر ClientProxy -> RpcException
اختيار الناقل         -> ضمان التسليم لا شهرته
```

## أبسط مثال

معالج فاتورة يرمي `RpcException` عند غياب الطلب. عميل `send` يدخل `catch` بعد `lastValueFrom` ويعرض تعذر الإصدار. لا يبحث عن حقل `statusCode` الخاص بـ HTTP إلا إذا وضعته أنت داخل كائن `RpcException`.

## شرح المثال

الكائن الصغير يكفي. لا ترسل صف Prisma ولا تتبع المكدس إلى عملية أخرى. المكدس يبقى في سجل المستمع. العميل يحصل على رسالة يفهمها.

## ماذا يحدث داخليًا؟

1. gRPC يطابق الأسلوب في الـ proto إلى الدالة.
2. MQTT يطابق الموضوع إلى `@EventPattern` أو نمط الرسالة حسب الصفحة.
3. `RpcException` تُلتقط في طبقة النقل وتُسلسل للعميل.
4. استثناء عادي قد يظهر كخطأ داخلي بلا جسم متفق عليه.
5. `HttpException` على مسار RPC لا تمر بفلتر HTTP إلا في الهجين وعلى مدخل HTTP.
6. العنوان في الخيارات يُحل داخل شبكة Docker باسم الخدمة.

## مثال عملي

فريق دفع خارجي يعطيك `billing.proto`. استعمل gRPC على تلك الحدود فقط. بين الكتالوج والفوترة داخل شركتك ابقِ على NATS أو Rabbit ما دام العقد يتغير. حساسات مخزن تتحدث MQTT إلى بوابة، والبوابة تترجم إلى أمر داخلي على طابور. لا توصل MQTT بقاعدة المتجر مباشرة من المعالج بلا تحقق.

## أخطاء شائعة

- gRPC لكل دالة داخلية متغيرة.
- MQTT كطابور فواتير.
- `throw new Error` بلا `RpcException` ثم شكل خطأ مختلف لكل ناقل.
- `localhost` في `url` بين الحاويات.
- تسريب مكدس داخل كائن الاستثناء.

## متى نستخدمه؟

gRPC لعقد protobuf مستقر بين فرق. MQTT لحافة أجهزة. `RpcException` كلما عبر الخطأ `ClientProxy`.

## متى لا نستخدمه؟

لا تجمع كل النواقل في العملية «للاحتياط». ناقل واحد واضح. ولا gRPC بديلًا عن حد وحدة لم يُرسم.

## مقارنة

| النقل | يناسبه | الحزمة الشائعة |
| --- | --- | --- |
| gRPC | عقد proto | `@grpc/grpc-js` و `@grpc/proto-loader` |
| MQTT | مواضيع خفيفة | `mqtt` |
| NATS | طلب-رد تشغيلي | `nats` |
| RabbitMQ | طابور أعمال | `amqplib` |
| Kafka | سجل أحداث | `kafkajs` |

## تمارين

### تمرين سهل

أي استثناء ترميه ليراه `send` على العميل؟

<details>
<summary>الحل</summary>

`RpcException` من `@nestjs/microservices`.

</details>

### تمرين متوسط

لماذا لا تضع المكدس في كائن `RpcException`؟

<details>
<summary>الحل</summary>

الرسالة تعبر عملية أخرى وقد تصل إلى سجل العميل. التفاصيل تبقى عند المستمع. العميل يأخذ سببًا قصيرًا.

</details>

### تمرين متقدم

حساسات MQTT وفريق دفع عنده proto. كم ناقلًا عند حدودك، وأين لا تضع منطق الخصم؟

<details>
<summary>الحل</summary>

MQTT عند الحساسات و gRPC عند حدود الدفع، أو ترجمة الحساسات إلى ناقل داخلي واحد. الخصم في خدمة الدومين لا في محول الموضوع ولا في ملف proto.

</details>

## أسئلة مراجعة

- ماذا يحتاج gRPC غير `Transport`؟
- ماذا لا يُخزن MQTT نيابة عن فواتيرك؟
- أين يُترجم خطأ HTTP في الهجين؟

## الخلاصة

gRPC عقد صارم و MQTT مواضيع خفيفة. على كل ناقل ارمِ `RpcException` للخطأ الذي يراه العميل، واختر النقل بالضمان، وحل العنوان باسم الخدمة داخل Compose.

## ماذا بعد؟

أمثلة TCP و NATS وطابور، ثم Docker حيث تعيش هذه العناوين.

## المصادر الرسمية

- NestJS — gRPC: https://docs.nestjs.com/microservices/grpc
- NestJS — MQTT: https://docs.nestjs.com/microservices/mqtt
- NestJS — Microservices: https://docs.nestjs.com/microservices/basics
- NestJS — Exception filters (RpcException): https://docs.nestjs.com/microservices/exception-filters
