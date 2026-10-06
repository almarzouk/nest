# Lifecycle Hooks

## الهدف من هذا الدرس

تنفيذ `afterInit` و `handleConnection` و `handleDisconnect` لتنظيف الغرف ومعرفة متى يصبح `server` جاهزًا.

## المشكلة التي يحلها

العميل يغلق اللابتوب بلا حدث `leave`. الغرفة ما زالت تحتفظ بمعرّف مقبس لن يسمع. والبث من `announce` في أول ميلي ثانية من الإقلاع قد يرى `server` غير مهيأ. الخطافات هي أماكن هذه اللحظات، لا منطق التعليق نفسه.

## الفكرة ببساطة

ثلاث واجهات: `OnGatewayInit` بعد بناء الخادم، `OnGatewayConnection` لكل اتصال، `OnGatewayDisconnect` عند القطع. الأخيرة أنسب مكان لتسجيل أن المقبس خرج. Socket.IO يزيله من الغرف عند القطع أصلًا، لكن أي `Map` بنيتها أنت تُحدَّث هنا.

## التعريف التقني

الصفحة تسمي الخطافات:

| الواجهة | الدالة | متى |
| --- | --- | --- |
| `OnGatewayInit` | `afterInit(server)` | بعد إنشاء خادم المقبس |
| `OnGatewayConnection` | `handleConnection(client, ...args)` | عند اتصال عميل |
| `OnGatewayDisconnect` | `handleDisconnect(client)` | عند انقطاعه |

`server` في `afterInit` هو نفسه الذي يربطه `@WebSocketServer`. المعامل `client` من نوع مقبس المكتبة، هنا `Socket`.

```ts
import {
  OnGatewayConnection,
  OnGatewayDisconnect,
  OnGatewayInit,
  WebSocketGateway,
} from '@nestjs/websockets';
import type { Server, Socket } from 'socket.io';

@WebSocketGateway({ namespace: 'comments' })
export class CommentsGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  afterInit(server: Server) {
    server.emit('ready');
  }

  handleConnection(client: Socket) {
    client.data.connectedAt = Date.now();
  }

  handleDisconnect(client: Socket) {
    client.data.connectedAt = undefined;
  }
}
```

`client.data` كيس على هذا المقبس فقط. لا تخزن فيه صف Prisma.

## Mental Model

```text
إقلاع -> afterInit
كل تبويب -> handleConnection
إغلاق التبويب أو انقطاع الشبكة -> handleDisconnect
```

الرسائل `@SubscribeMessage` تقع بين الاتصال والقطع.

## أبسط مثال

عدّاد `live` يزيد في `handleConnection` وينقص في `handleDisconnect`. الرقم التجميعي على البوابة مقبول. مصفوفة نصوص تعليقات بجانبه ليست كذلك.

## شرح المثال

`afterInit` الذي ينادي `server.emit('ready')` يصل فقط إلى من اتصل قبل ذلك، أي لا أحد غالبًا. لا تستخدمه للبث المنتج. استخدمه لتسجيل أن الفضاء جاهز أو لضبط وسيط مكتبة. العدّاد يوضح أن الخطاف يرى كل المقابس بلا `@SubscribeMessage`.

## ماذا يحدث داخليًا؟

المحرك ينادي `afterInit` مرة ويمرر الخادم. كل مصافحة ناجحة تنادي `handleConnection` قبل أن تعتمد على رسائل تطبيقك. القطع، سواء أغلق العميل أو انقطع المهلة، ينادي `handleDisconnect`. غرف Socket.IO تُحدَّث مع القطع من المكتبة. إن نسخت الأعضاء في `Map<bookId, Set<socketId>>` فاحذف `socket.id` هنا وإلا كبر السجل. لا تستدعِ قاعدة بيانات بطيئة في `handleConnection` قبل أن تقرر إن كان الاتصال مقبولًا. الفحص الثقيل يؤخر المصافحة.

## مثال عملي

سجّل `client.id` عند الاتصال والقطع. هو معرف مكتبة لا معرف مستخدم. اربط `userId` بعد فحص الرمز في درس المصادقة، واحذف الربط في `handleDisconnect`.

## أخطاء شائعة

- بث منتج داخل `afterInit`.
- الاعتماد على رسالة `leave` وحدها لتنظيف `Map` محلية.
- عمل قاعدة متسلسل في `handleConnection` لكل تبويب.
- حفظ `client` في مصفوفة عامة وعدم حذفه عند القطع.

## متى نستخدمه؟

`afterInit` لضبط الخادم مرة. الاتصال والقطع للمحاسبة والتنظيف ورفض مبكر ستكمله المصادقة.

## متى لا نستخدمه؟

لتنفيذ `join`. ذلك رسالة يرسلها العميل عندما يعرف `bookId`. الخطاف لا يعرف أي كتاب فُتحت صفحته إلا إذا وضعته في المصافحة.

## مقارنة

| اللحظة | الخطاف | ليس مكانه |
| --- | --- | --- |
| الخادم جاهز | `afterInit` | بث تعليق |
| تبويب فُتح | `handleConnection` | معرفة الكتاب تلقائيًا |
| التبويب مات | `handleDisconnect` | حفظ تعليق |

## تمارين

### تمرين سهل

أي دالة تُستدعى مرة بعد بناء الخادم؟

<details>
<summary>الحل</summary>

`afterInit` عبر `OnGatewayInit`.

</details>

### تمرين متوسط

لماذا لا يكفي حدث `leave` لتنظيف بنية تملكها أنت؟

<details>
<summary>الحل</summary>

لأن الإغلاق المفاجئ لا يرسل `leave`. `handleDisconnect` يعمل عند انقطاع المقبس.

</details>

### تمرين متقدم

ماذا يجوز أن يبقى على البوابة بين الاتصالات، وماذا لا يجوز؟

<details>
<summary>الحل</summary>

عدّاد أو بنية تُحذف مفاتيحها عند القطع. لا نص مستخدم ولا كائن طلب HTTP ولا عميل بلا حذف.

</details>

## أسئلة مراجعة

- ما الواجهات الثلاث؟
- متى تُملأ خاصية `@WebSocketServer` نسبة إلى الباني؟
- ماذا يفعل Socket.IO بالغرف عند القطع؟

## الخلاصة

`afterInit` مرة، والاتصال والقطع لكل مقبس. نظّف ما بنيته يدويًا في `handleDisconnect`. الانضمام إلى كتاب يبقى رسالة.

## ماذا بعد؟

رفض المقبس الذي بلا رمز، في `handleConnection` قبل أي `join`.

## المصادر الرسمية

- NestJS — Gateways: https://docs.nestjs.com/websockets/gateways
