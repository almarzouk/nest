# Socket.IO

## الهدف من هذا الدرس

تثبيت أن بوابة Nest الافتراضية تتحدث Socket.IO: أسماء الغرف، وعميل المتصفح، وما الذي ليس WebSocket الخام.

## المشكلة التي يحلها

عميل `new WebSocket('ws://...')` يرسل إطارات خام. البوابة التي كتبتها تستعمل `client.join` و `emit('join', payload)`. هذه واجهة Socket.IO فوق النقل، وفيها مصافحة خاصة ومسار `/socket.io`. العميل الخام لا يكمل المصافحة، فتظن أن البوابة معطلة.

## الفكرة ببساطة

Nest، ما لم تبدّل المحول، تستخدم منصة Socket.IO. الخادم حزمة `socket.io` والعميل `socket.io-client`. الغرفة والبث وacknowledgment التي استعملتها في الدروس السابقة من هذه المكتبة. منصة `ws` بديل أنحف بلا غرف جاهزة، وتحتاج محولًا صريحًا. هذه الدورة تبقى على الافتراض.

## التعريف التقني

تثبيت شائع بجانب Nest:

```bash
npm i @nestjs/platform-socket.io socket.io
npm i -D socket.io-client
```

`@nestjs/platform-socket.io` يأتي غالبًا مع حزم المنصة. إن غاب `Server` من الاستيراد فالحزمة غير مثبتة.

عميل الواجهة:

```ts
import { io } from 'socket.io-client';

export const comments = io('http://localhost:3000/comments', {
  autoConnect: false,
});
```

المسار `/comments` هو `namespace` في الزينة، لا مسار REST. `autoConnect: false` حتى لا تُفتح المصافحة قبل أن تملك الواجهة ما يلزم، ثم `comments.connect()`.

إرسال واستقبال:

```ts
comments.emit('join', { bookId: '42' }, (ack: { joined: string }) => {
  console.log(ack.joined);
});
comments.on('comment', (payload: { id: string; text: string }) => {
  console.log(payload.text);
});
```

الدالة الأخيرة في `emit` تستلم إقرار `return` من البوابة.

## Mental Model

```text
socket.io-client
  namespace /comments
    emit join -> SubscribeMessage
    on comment <- server.to(room).emit
```

## أبسط مثال

الانضمام عندما تُفتح صفحة الكتاب، والقطع عند مغادرتها:

```ts
export function watchBook(bookId: string) {
  comments.connect();
  comments.emit('join', { bookId });
  return () => {
    comments.emit('leave', { bookId });
    comments.disconnect();
  };
}
```

`leave` يحتاج `@SubscribeMessage('leave')` ينادي `client.leave(bookId)`. من غير المغادرة تبقى الغرفة تحتفظ بمقبس صفحة أُغلقت إن لم ينقطع الاتصال.

## شرح المثال

الاتصال واحد لفضاء التعليقات، والانضمام يتغير مع معرف الكتاب. عند الانتقال من كتاب 42 إلى كتاب 7، غادر الغرفة الأولى ثم انضم إلى الثانية على المقبس نفسه. قطع الاتصال لكل تنقل يعمل أيضًا لكنه أغلى. `console.log` هنا مكانه مخزن الواجهة لا الخادم.

## ماذا يحدث داخليًا؟

عميل Socket.IO يبدأ طلبًا إلى مسار المكتبة، ويترقّى إلى نقل طويل العمر (WebSocket غالبًا، مع بدائل داخل المكتبة إن فشل النقل). الرسالة `emit` إطار باسم. الخادم يوزع الاسم على `@SubscribeMessage`. `to(room)` ذاكرة الغرف داخل عملية Node التي تحتضن الخادم. عميل `ws` الخام لا يرسل تنسيق Socket.IO، فيرفض الخادم الجلسة قبل `join`.

تغيير المحول إلى `ws` يلغي `client.join` ما لم تبنِ الغرف. لا تخلط مثال `ws` من مقال مع بوابة لم تغيّر `WsAdapter`.

## مثال عملي

تحقق سريع من الطرفية باستخدام عميل Socket.IO لا `wscat` وحده:

1. شغّل Nest على 3000.
2. من الواجهة أو سكربت `io('http://localhost:3000/comments')`.
3. أرسل `join` لكتاب معروف.
4. `POST` تعليقًا وتوقع حدث `comment` في العميل المنضم فقط.

إن نجح `wscat` وفشل هذا، فأنت تختبر قناة أخرى غير البوابة.

## أخطاء شائعة

- عميل `WebSocket` خام ضد بوابة Socket.IO.
- الاتصال بـ `http://localhost:3000` ونسيان فضاء `/comments`.
- البقاء في الغرفة بعد تغيير صفحة الكتاب.
- افتراض أن Socket.IO يشارك الغرف بين عمليتَي Node.

## متى نستخدمه؟

ما دامت البوابة على المحول الافتراضي. العميل الرسمي `socket.io-client` بنفس إصدار الجيل المتوافق مع الخادم.

## متى لا نستخدمه؟

عندما تحتاج إطارات خام قليلة بلا غرف وتبدّل المحول إلى `ws` عن قصد. عندها تُعاد كتابة `join` و `emit`. لا تخلط العميلين على البوابة نفسها.

## مقارنة

| العميل | هل يكلم البوابة الافتراضية |
| --- | --- |
| `socket.io-client` إلى الفضاء الصحيح | نعم |
| `new WebSocket` | لا، التنسيق مختلف |
| متصفح GraphiQL | لا، ذلك اشتراك GraphQL |

## تمارين

### تمرين سهل

أي حزمة في الواجهة تتصل بالبوابة الافتراضية؟

<details>
<summary>الحل</summary>
`socket.io-client`، إلى فضاء الاسم مثل `/comments`.
</details>

### تمرين متوسط

أين يصل رد `return { joined }` في العميل؟

<details>
<summary>الحل</summary>
في دالة الإقرار، الوسيط الأخير لـ `emit('join', body, ack)`.
</details>

### تمرين متقدم

لماذا مغادرة الغرفة عند تغيير الكتاب جزء من العقد؟

<details>
<summary>الحل</summary>
لأن المقبس إن بقي في غرفة 42 يصلحه تعليق ذلك الكتاب وهو يعرض الكتاب 7.
</details>

## أسئلة مراجعة

- ما مسار المصافحة مقارنة بمسار REST؟
- ماذا يوفر Socket.IO فوق المقبس الخام في هذه الدروس؟
- أي أداة لا تثبت أن البوابة تعمل؟

## الخلاصة

الافتراض Socket.IO. العميل من العائلة نفسها، والفضاء يطابق الزينة، والغرفة تُغادر عند تغيير الصفحة.

## ماذا بعد؟

خطافات الإقلاع والاتصال والقطع، حيث تُنظَّف الغرفة حتى لو نسي العميل `leave`.

## المصادر الرسمية

- NestJS — Gateways: https://docs.nestjs.com/websockets/gateways
- Socket.IO — Rooms: https://socket.io/docs/v4/rooms/
