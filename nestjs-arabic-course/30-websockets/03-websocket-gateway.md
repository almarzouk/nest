# @WebSocketGateway

## الهدف من هذا الدرس

ضبط الزينة: المنفذ، وفضاء الاسم، وCORS للمتصفح، والوصول إلى خادم المقبس.

## المشكلة التي يحلها

بوابة بلا خيارات تسمع على منفذ التطبيق وفضاء الاسم الافتراضي. الواجهة على `localhost:5173` يرفضها المتصفح لأن مصافحة المقبس تخضع لـ CORS. وفصل المقبس على منفذ آخر بلا قصد يكسر عنوانًا افترضه العميل على `/socket.io` بجانب API.

## الفكرة ببساطة

وسيط الزينة الأول رقم منفذ اختياري. الوسيط الآخر كائن خيارات يصل إلى مكتبة المقبس. للتطوير المحلي مع واجهة منفصلة ضع `cors.origin` على أصل الواجهة. `@WebSocketServer()` يعطيك الخادم لبث لاحق لا يخص مرسلًا واحدًا.

## التعريف التقني

الصفحة: المنفذ الافتراضي هو منفذ HTTP. `@WebSocketGateway(80)` يغيّره. فضاء الاسم:

```ts
@WebSocketGateway({ namespace: 'comments' })
```

CORS في كائن الخيارات لأن Socket.IO يتحقق منه أثناء المصافحة:

```ts
@WebSocketGateway({
  namespace: 'comments',
  cors: { origin: ['http://localhost:5173'], credentials: true },
})
```

`credentials` إن كان العميل يرسل ملف تعريف ارتباط. لا تفتح `origin: '*'` مع `credentials`. في الإنتاج الأصل نطاق الواجهة الحقيقي لا لوكال المضيف.

`@WebSocketServer()` خاصية تمتلئ بالخادم بعد الإقلاع. نوعها من `socket.io` عندما يكون المحول الافتراضي: `Server`.

```ts
import { WebSocketServer } from '@nestjs/websockets';
import type { Server } from 'socket.io';

@WebSocketServer()
server: Server;
```

لا تستخدم الخاصية في الباني. هي جاهزة بعد تهيئة البوابة، أي في الخطاف `afterInit` أو عند أول رسالة.

## Mental Model

```text
تطبيق Nest على :3000
  namespace /comments
    CORS origin الواجهة
      server.emit أو server.to(room)
```

## أبسط مثال

```ts
import { WebSocketGateway, WebSocketServer } from '@nestjs/websockets';
import type { Server } from 'socket.io';

@WebSocketGateway({
  namespace: 'comments',
  cors: { origin: ['http://localhost:5173'] },
})
export class CommentsGateway {
  @WebSocketServer()
  server: Server;

  announce(bookId: string, body: string) {
    this.server.to(bookId).emit('comment', { bookId, body });
  }
}
```

`announce` دالة عادية يستدعيها المتحكم بعد الحفظ. `to(bookId)` غرفة. الانضمام إليها رسالة لاحقة، لا أثر لهذه الدالة وحدها.

## شرح المثال

الواجهة تتصل بمساحة `comments` على أصل الخادم، مثل `http://localhost:3000/comments` حسب عميل Socket.IO. `cors.origin` يجب أن يطابق أصل الصفحة بما فيه المنفذ. `announce` لا تعرف HTTP. هي تبث إلى غرفة باسم معرف الكتاب. من لم ينضم لا يسمع، وهذا مقصود.

## ماذا يحدث داخليًا؟

أثناء الإقلاع يبني المحول خادم Socket.IO ويمرر خيارات الزينة، ومنها CORS وفضاء الاسم. `@WebSocketServer` يربط الخاصية بتلك النسخة. `server.to(room).emit` يكتب إطارًا لكل مقبس في الغرفة على هذه العملية. الغرفة فارغة حتى ينادي عميل `join`. المنفذ المختلف عن HTTP يعني أن العميل لا يفترض عنوان REST نفسه، وأن الجدار الناري يفتح المنفذين.

## مثال عملي

إنتاج خلف نطاق واحد `https://books.example` يخدم الواجهة والـ API. عندها قد لا تحتاج أصلًا مختلفًا. إن بقيت الواجهة على نطاق آخر، عدّل `origin` من إعداد لا من نص ثابت في المستودع:

```ts
cors: { origin: [process.env.WEB_ORIGIN ?? 'http://localhost:5173'] },
```

تحقق في المتصفح من خطأ CORS على طلب المصافحة قبل أن تلوم `@SubscribeMessage`. المصافحة تفشل قبل أي رسالة تطبيق.

## أخطاء شائعة

- `origin: '*'` مع اعتماد على ملفات الارتباط.
- قراءة `this.server` من الباني.
- بث `server.emit` إلى كل الفضاء بدل `to(bookId)`.
- تغيير المنفذ في الزينة وإبقاء عميل الواجهة على منفذ REST.

## متى نستخدمه؟

على كل بوابة سيصلها متصفح من أصل آخر، ومع أي بث خارج معالج الرسالة.

## متى لا نستخدمه؟

لا تفصل المنفذ بلا سبب تشغيلي. المنفذ المشترك مع HTTP أبسط خلف وكيل واحد. ولا تضع أسرار CORS في كود يُنشر لكل البيئات إن كان الأصل يختلف.

## مقارنة

| الخيار | إن نُسي |
| --- | --- |
| `namespace` | تختلط الأحداث مع بوابة أخرى على الجذر |
| `cors.origin` | المتصفح يوقف المصافحة |
| `@WebSocketServer` | لا بث إلا لمن أعاد الدالة ردًا له وحده |

## تمارين

### تمرين سهل

أي زينة تحصل على كائن `Server`؟

<details>
<summary>الحل</summary>

`@WebSocketServer()` على خاصية في البوابة.

</details>

### تمرين متوسط

لماذا تفشل الواجهة على منفذ 5173 قبل إرسال تعليق؟

<details>
<summary>الحل</summary>

مصافحة Socket.IO ترفض أصل الصفحة إن لم يكن في `cors.origin`.

</details>

### تمرين متقدم

متى تستدعي `announce` نسبة إلى حفظ الصف؟

<details>
<summary>الحل</summary>

بعد نجاح الحفظ. البث قبل الحفظ يُظهر تعليقًا لا يعيده الجلب.

</details>

## أسئلة مراجعة

- ما منفذ البوابة الافتراضي؟
- ماذا يفعل `server.to(bookId)`؟
- لماذا لا تُقرأ `server` في الباني؟

## الخلاصة

`@WebSocketGateway` يحدد الفضاء وCORS والمنفذ. `@WebSocketServer` للبث إلى غرفة بعد أن تكون الخاصية جاهزة.

## ماذا بعد؟

`@SubscribeMessage`: استقبال `join` و `comment` من العميل.

## المصادر الرسمية

- NestJS — Gateways: https://docs.nestjs.com/websockets/gateways
