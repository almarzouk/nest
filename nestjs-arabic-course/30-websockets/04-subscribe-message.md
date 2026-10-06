# @SubscribeMessage

## الهدف من هذا الدرس

ربط اسم رسالة بدالة، وقراءة الجسم والمقبس، ومعرفة أن قيمة الإرجاع إيصال للمرسل لا بثًا للغرفة.

## المشكلة التي يحلها

بعد ضبط البوابة لا يوجد بعد اسم يستمع إليه الخادم. العميل يرسل `join` والخادم يتجاهله. أو الدالة تعيد التعليق فتصل للمرسل فقط، والقارئ الآخر لا يراه، لأن الإرجاع ليس `emit` للغرفة.

## الفكرة ببساطة

`@SubscribeMessage('join')` يربط الاسم. `@MessageBody()` جسم الرسالة. `@ConnectedSocket()` مقبس هذا العميل. أعد قيمة صغيرة كإيصال. للآخرين استخدم `server.to(...).emit` أو `client.to(room).emit`.

## التعريف التقني

```ts
import {
  ConnectedSocket,
  MessageBody,
  SubscribeMessage,
  WebSocketGateway,
  WebSocketServer,
} from '@nestjs/websockets';
import type { Server, Socket } from 'socket.io';

@WebSocketGateway({ namespace: 'comments' })
export class CommentsGateway {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('join')
  join(
    @MessageBody() body: { bookId: string },
    @ConnectedSocket() client: Socket,
  ) {
    client.join(body.bookId);
    return { joined: body.bookId };
  }
}
```

`client.join` غرفة Socket.IO باسم معرف الكتاب. الإرجاع `{ joined }` يذهب كإقرار إلى المرسل إن كان العميل ينتظر ack. بقية الغرفة لا تستلم هذا الكائن.

التحقق من شكل `body` أنبوب لاحق. هنا افترض كائنًا بسيطًا. لا تثق بـ `bookId` القادم من العميل كمسار قرص. هو مفتاح غرفة ونص يُفحص مقابل الكتب إن لزم.

## Mental Model

```text
عميل emit('join', { bookId })
  join()
    client.join(bookId)
    return إيصال للمرسل فقط
عميل emit('comment', payload) من مسار آخر
  server.to(bookId).emit('comment', payload)
```

## أبسط مثال

رسالة لا تحفظ في القاعدة، فقط ترفض البث المباشر من المتصفح:

```ts
@SubscribeMessage('comment')
comment(@MessageBody() body: { bookId: string; text: string }) {
  return { accepted: false, reason: 'use POST /books/:id/comments' };
}
```

الحفظ عبر HTTP حتى يبقى التحقق والقاعدة على المسار الذي اختبرته. البوابة للانضمام وللبث الذي يستدعيه المتحكم عبر `announce`. السماح للعميل ببث `comment` مباشرة يتخطى الخدمة.

## شرح المثال

هذا الرفض مقصود في تصميم المتجر. رسالة `join` ضرورية حتى تمتلئ الغرفة. رسالة `comment` على المقبس ليست باب الكتابة. إن أعاد العميل الإرسال ظنًا أن المقبس يحفظ، الإيصال `{ accepted: false }` يشرح العقد. البث الحقيقي:

```ts
announce(bookId: string, comment: { id: string; text: string }) {
  this.server.to(bookId).emit('comment', comment);
}
```

والمتحكم بعد `201` ينادي `announce`.

## ماذا يحدث داخليًا؟

Socket.IO يمرر اسم الحدث إلى الدالة المسجّلة. Nest تملأ المعاملات من الجسم والمقبس. قيمة الإرجاع تُرسل كرد على الحدث للعميل الذي أرسل، وهو نمط acknowledgment. `emit` على `server` حدث جديد باسم قد يختلف عن اسم الرسالة الواردة. المستمع في الواجهة `socket.on('comment', ...)` لا يرى قيمة `return`. من لم ينفّذ `join` ليس في الغرفة فلا يصله `to(bookId)`.

## مثال عملي

عقد الواجهة:

| الحدث الصادر | من يعالجه | النتيجة |
| --- | --- | --- |
| `join` | البوابة | الغرفة + إيصال |
| `POST` تعليق | المتحكم والخدمة | صف |
| `comment` الوارد من الخادم | الواجهة | إلحاق بالقائمة |

لا تسمِّ إيصال الانضمام `comment` حتى لا ترسمه الواجهة كتعليق.

## أخطاء شائعة

- اعتبار `return` بثًا لكل الغرفة.
- قبول `comment` من المقبس وكتابته في القاعدة بلا الأنابيب التي على REST.
- `client.join` بمعرف لم يُفحص، فيصنع العميل غرفًا بلا نهاية.
- نسيان أن الإقرار لا يصل إلى `@SubscribeMessage` آخر.

## متى نستخدمه؟

لرسائل المقبس الحقيقية: انضمام، مغادرة، مؤشر. الكتابة الدائمة تبقى مسارًا يستدعي `announce`.

## متى لا نستخدمه؟

لكل عملية لها DTO على HTTP. تكرار العقدين يضاعف التحقق. رسالة واحدة للانضمام أوضح من إعادة تنفيذ `POST` داخل البوابة.

## مقارنة

| الإجراء | من يسمع |
| --- | --- |
| `return` من المعالج | المرسل، كإقرار |
| `server.to(room).emit` | المنضمون إلى الغرفة |
| `server.emit` | كل متصل بالفضاء على هذه العملية |

## تمارين

### تمرين سهل

أي زينة تربط الدالة باسم `join`؟

<details>
<summary>الحل</summary>
`@SubscribeMessage('join')`.
</details>

### تمرين متوسط

لماذا لا يظهر `return { joined }` عند القارئ الآخر؟

<details>
<summary>الحل</summary>
لأن الإرجاع إقرار للمرسل. البث للغرفة `emit` منفصل.
</details>

### تمرين متقدم

لماذا يرفض المثال حفظ التعليق القادم من المقبس؟

<details>
<summary>الحل</summary>
حتى يبقى التحقق والحفظ على `POST`، والبوابة تنضم وتبث بعد النجاح فقط.
</details>

## أسئلة مراجعة

- ماذا يفعل `client.join`؟
- من يستدعي `announce`؟
- أي اسم حدث يجب ألا يختلط بالإقرار؟

## الخلاصة

`@SubscribeMessage` يستقبل حدثًا. الإرجاع للمرسل. الغرفة تُملأ بـ `join`، والتعليق المحفوظ يُبث بـ `emit` بعد المسار.

## ماذا بعد؟

Socket.IO هو المحول الذي جعل `join` و `to` و `emit` بهذه الأسماء.

## المصادر الرسمية

- NestJS — Gateways: https://docs.nestjs.com/websockets/gateways
