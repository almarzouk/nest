# Authentication

## الهدف من هذا الدرس

فحص الرمز عند الاتصال وإغلاق المقبس قبل `join`، لأن حارس الرسالة يأتي بعد أن صار العميل متصلًا.

## المشكلة التي يحلها

`@UseGuards` على `@SubscribeMessage` لا يمنع المصافحة. عميل بلا رمز يبقى في الفضاء ويستمع إن انضم إلى غرفة عامة أو إن وصلته أحداث على مستوى الفضاء. صفحة الكتاب قد تكون عامة، لكن بث المسودات أو إشعار المحرر ليس عامًا. الانتظار حتى أول رسالة يترك نافذة استماع.

## الفكرة ببساطة

العميل يمرر الرمز في مصافحة Socket.IO: `auth: { token }` لا في عنوان يُسجَّل بسهولة. `handleConnection` يتحقق. عند الفشل `client.disconnect()`. عند النجاح ضع `userId` على `client.data`. حارس الرسالة يبقى للعمليات التي تحتاج الدور، لا بديلًا عن إغلاق الاتصال.

## التعريف التقني

عميل:

```ts
import { io } from 'socket.io-client';

io('http://localhost:3000/comments', {
  auth: { token: 'Bearer eyJ...' },
});
```

البوابة:

```ts
import type { Socket } from 'socket.io';

handleConnection(client: Socket) {
  const token = client.handshake.auth?.token;
  const userId = this.tokens.verify(String(token ?? ''));
  if (!userId) {
    client.disconnect();
    return;
  }
  client.data.userId = userId;
}
```

`handshake.auth` حمولة مصافحة Socket.IO. `handshake.query` يظهر في العنوان والسجلات، فتجنّبه للرمز. `verify` تعيد معرفًا أو فراغًا. تفاصيل JWT نفسها من فصل المصادقة في الدورة، لا تُعاد هنا. هذه الدالة لا تثق بالسلسلة.

`client.data` متاح لاحقًا:

```ts
@SubscribeMessage('join')
join(@ConnectedSocket() client: Socket, @MessageBody() body: { bookId: string }) {
  return { joined: body.bookId, as: client.data.userId };
}
```

## Mental Model

```text
مصافحة auth.token
  verify فشل -> disconnect
  verify نجح -> data.userId
    بعدها join
```

## أبسط مثال

تعليقات الكتاب المنشور قد تسمح باتصال بلا حساب للقراءة فقط. افصل الفضاءات إن اختلفت السياسة:

```text
/comments   كتالوج منشور، join بلا رمز
/desk       مسودات المحرر، handleConnection يقطع بلا رمز
```

لا تضع السياستين على فضاء واحد ثم تنسَ فحص الغرفة. غرفة `draft:42` على فضاء عام تعني أن من عرف الاسم سمع.

## شرح المثال

الفضاء العام لا يستدعي `verify`. فضاء المكتب يستدعيه دائمًا. `disconnect` قبل أي `emit` منتج. إن رغبت برسالة خطأ قبل القطع، أرسل حدثًا ثم افصل في المهلة التالية. القطع الصامت أسهل للعقد: الواجهة ترى `disconnect` وتعيد المستخدم إلى الدخول.

## ماذا يحدث داخليًا؟

Socket.IO يضع `auth` على كائن المصافحة لهذه الجلسة. `handleConnection` يعمل بعد نجاح النقل وقبل رسائل تطبيقك إن كتبت الفحص في أوله. `disconnect` ينهي الجلسة ويستدعي `handleDisconnect`. من لم يُقطع يستطيع `emit('join')` إلى أي `bookId` يعرفه. لذلك المصادقة تجيب «من هذا»، ورسالة `join` ما زالت تحتاج قرار «هل يرى هذا الكتاب». معرف في `client.data` لا يغني عن فحص العضوية إن كانت المسودات خاصة. حارس `@SubscribeMessage` يرى المقبس عبر `switchToWs().getClient()` ويمنع حدثًا واحدًا. الاتصال نفسه قرار الخطاف.

## مثال عملي

`join` لمسودة:

```ts
@SubscribeMessage('join')
join(@ConnectedSocket() client: Socket, @MessageBody() body: { bookId: string }) {
  const allowed = this.books.canReadDraft(body.bookId, client.data.userId);
  if (!allowed) {
    return { joined: null };
  }
  client.join(`draft:${body.bookId}`);
  return { joined: body.bookId };
}
```

المستخدم معروف من المصافحة. الإذن على الكتاب من الخدمة. الإرجاع `{ joined: null }` إيصال رفض بلا بث.

## أخطاء شائعة

- الرمز في `query` لأنه أسهل في التجربة.
- حارس على `comment` فقط، والعميل يسمع `server.emit` العام.
- الثقة بـ `body.userId` من رسالة `join`.
- إبقاء الاتصال بعد فشل `verify` مع حدث `error` بلا `disconnect`.

## متى نستخدمه؟

على أي فضاء ليس عامًا بالكامل، وفي أول `handleConnection`.

## متى لا نستخدمه؟

لفضاء تعليقات الكتالوج المنشور إن كانت الغرف لا تحمل مسودات. التعقيد هناك يقطع القارئ بلا مكسب. لا تعِد فحص كلمة المرور في كل رسالة إن كان الرمز فُحص عند الاتصال وما زال الاتصال قصير العمر.

## مقارنة

| المكان | ماذا يمنع |
| --- | --- |
| `handleConnection` | بقاء مقبس بلا هوية |
| حارس على الرسالة | حدثًا واحدًا بعد الاتصال |
| `join` + خدمة | دخول غرفة بلا إذن على المورد |

## تمارين

### تمرين سهل

أين يضع عميل Socket.IO الرمز في هذا الدرس؟

<details>
<summary>الحل</summary>
في `auth: { token }` عند إنشاء الاتصال، ويُقرأ من `handshake.auth`.
</details>

### تمرين متوسط

لماذا الحارس على الرسالة لا يغلق مستمعًا صامتًا؟

<details>
<summary>الحل</summary>
لأنه لا يعمل إلا عند رسالة مزينة. الاتصال والبث العام يسبقان ذلك.
</details>

### تمرين متقدم

المستخدم مصادق وانضم إلى `draft:42` وهو ليس المحرر. أي فحص نقص؟

<details>
<summary>الحل</summary>
فحص العضوية في `join` عبر الخدمة. الهوية تعرف من المتصل، ولا تثبت أنه يملك المسودة.
</details>

## أسئلة مراجعة

- ماذا تستدعي عند فشل `verify`؟
- لماذا تُرفض `body.userId`؟
- أي فضاء في المثال يبقى بلا رمز؟

## الخلاصة

افحص الرمز في `handleConnection` واقطع الفشل. ضع `userId` على `client.data`. إذن المورد يبقى في `join` عبر الخدمة.

## ماذا بعد؟

الأنبوب والمرشح والحارس على الرسالة نفسها، بعد أن صار الاتصال مقبولًا.

## المصادر الرسمية

- NestJS — Gateways: https://docs.nestjs.com/websockets/gateways
- NestJS — Guards: https://docs.nestjs.com/guards
