# Filters و Pipes و Guards على WebSocket

## الهدف من هذا الدرس

تطبيق أنبوب على جسم الرسالة، وحارس على حدث واحد، ومرشح يحوّل الفشل إلى خطأ مقبس لا إلى صفحة HTTP.

## المشكلة التي يحلها

`join` يستقبل `bookId` من العميل. من غير أنبوب يصل `{}` أو رقم وتُبنى غرفة باسم `undefined`. الحارس على الاتصال أثبت الهوية، لا شكل الرسالة. وإن رميت `BadRequestException` فقد لا يراها عميل Socket.IO كما يراها `fetch`، لأن لا رمز حالة HTTP على الإطار.

## الفكرة ببساطة

`ValidationPipe` على `@MessageBody()` يفحص DTO. `@UseGuards` على الدالة يمنع الحدث. الاستثناء الخاص بالقناة `WsException`. مرشح `@Catch(WsException)` يرسل الحمولة إلى العميل. أبقِ الحفظ على HTTP. هنا الشكل والإذن على الرسالة فقط.

## التعريف التقني

DTO بـ `class-validator` كما في بقية الدورة:

```ts
import { IsString, Length } from 'class-validator';

export class JoinBookDto {
  @IsString()
  @Length(1, 64)
  bookId: string;
}
```

```ts
import { UsePipes, ValidationPipe } from '@nestjs/common';
import { MessageBody, SubscribeMessage, WsException } from '@nestjs/websockets';

@SubscribeMessage('join')
@UsePipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
join(@MessageBody() body: JoinBookDto) {
  return { joined: body.bookId };
}
```

`whitelist` يسقط الحقول الزائدة حتى لا يمر `userId` داخل الجسم إلى منطق الغرفة.

حارس حدث، بعد أن وضع الاتصال `userId`:

```ts
@Injectable()
export class WsUserGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const client = context.switchToWs().getClient<{ data: { userId?: string } }>();
    if (!client.data.userId) {
      throw new WsException('unauthenticated');
    }
    return true;
  }
}
```

مرشح يلتقط `WsException` ويرسل حدث خطأ. ابنِ من `BaseWsExceptionFilter` إن أردت السلوك الافتراضي ثم عدّل الحمولة. الحد الأدنى أن العميل يرى `{ status: 'error', message }` على الإقرار أو على حدث تعرفه الواجهة، لا HTML.

## Mental Model

```text
رسالة join
  Guard  هل في data.userId؟
  Pipe   هل JoinBookDto صالح؟
  الدالة
  WsException -> مرشح -> العميل
```

ترتيب Nest المعتاد: الحارس ثم الأنبوب ثم الدالة، والمرشح حولها.

## أبسط مثال

رفض حقل زائد:

```json
{ "bookId": "42", "userId": "attacker" }
```

مع `forbidNonWhitelisted: true` يفشل الأنبوب. الغرفة لا تُبنى، والهوية تبقى ما كتبه `handleConnection` في `client.data`.

## شرح المثال

الأنبوب نفسه الذي يحمي `POST` يحمي جسم المقبس إن ركّبته على الرسالة. الفرق قناة الخطأ: `WsException` تُترجم في مرشح على البوابة، و `BadRequestException` وحدها قد تُسجَّل ولا تصل كإقرار.

## ماذا يحدث داخليًا؟

عند وصول الحدث تبني Nest سياق `ws`. `switchToWs().getClient()` المقبس و `getData()` الجسم. الحارس الذي يعيد `false` يمنع الدالة. الذي يرمي `WsException` يذهب إلى المرشح. `ValidationPipe` ينشئ نسخة من `JoinBookDto` ويشغّل المدققات. الفشل استثناء قبل `client.join`. الإرجاع الناجح إقرار للمرسل فقط. على المقبس الجسم هو الرسالة، لا `request.query`.

## مثال عملي

جسم بلا `bookId` يفشل في المرشح. اتصال بلا `data.userId` يُقطع من الخطاف. رفض المسودة `{ joined: null }` من الدالة. ثلاث قنوات أوضح من رمز HTTP غير موجود هنا.

## أخطاء شائعة

- الاعتماد على `ValidationPipe` العالمي لـ HTTP دون وضعه على الرسالة.
- `throw new BadRequestException` وتوقع JSON المعتاد في المتصفح.
- حارس يقرأ `switchToHttp()` داخل البوابة.
- تمرير الجسم كاملًا إلى `client.join` قبل الأنبوب.

## متى نستخدمه؟

على كل `@SubscribeMessage` يقبل كائنًا من العميل، خصوصًا `join`.

## متى لا نستخدمه؟

بديلًا عن فحص الرمز في `handleConnection`. الأنبوب لا يعمل إن لم تُرسل رسالة. ولا بديلًا عن `ParseFilePipe` على الرفع.

## مقارنة

| الأداة | على HTTP | على البوابة |
| --- | --- | --- |
| الأنبوب | `@Body()` | `@MessageBody()` |
| الحارس | `switchToHttp` | `switchToWs` |
| خطأ العميل | `HttpException` | `WsException` ومرشح |

## تمارين

### تمرين سهل

أي استثناء يليق برسالة المقبس؟

<details>
<summary>الحل</summary>

`WsException` من `@nestjs/websockets`.

</details>

### تمرين متوسط

ماذا يفعل `forbidNonWhitelisted` على `join`؟

<details>
<summary>الحل</summary>

يرفض الجسم الذي يحمل حقولًا خارج `JoinBookDto`، مثل `userId` يحاول العميل تثبيته.

</details>

### تمرين متقدم

لماذا يبقى `handleConnection` لازمًا مع وجود `WsUserGuard`؟

<details>
<summary>الحل</summary>

الحارس يعمل عند الرسالة. الاتصال الصامت يسبقها. الخطاف يقطع من بلا رمز قبل الاستماع.

</details>

## أسئلة مراجعة

- أين يُركَّب `ValidationPipe` للرسالة؟
- كيف يصل الحارس إلى المقبس؟
- لماذا الإرجاع الناجح ليس بثًا؟

## الخلاصة

الأنبوب يشكّل `join`، والحارس يقرأ مقبس `ws`، و `WsException` يخرج من مرشح تفهمه الواجهة. الهوية المبكرة تبقى في خطاف الاتصال.

## ماذا بعد؟

RxJS الذي ستحتاجه المعترضات وعميل HTTP: `Observable` قبل أن نستدعي خدمة خارجية.

## المصادر الرسمية

- NestJS — Gateways: https://docs.nestjs.com/websockets/gateways
- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- NestJS — Pipes: https://docs.nestjs.com/pipes
