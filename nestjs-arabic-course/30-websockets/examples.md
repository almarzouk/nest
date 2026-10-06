# أمثلة عملية

تعليقات كتاب: الحفظ HTTP، والحضور مقبس Socket.IO على فضاء `comments`.

## مثال 1 — البوابة

```ts
import { UseFilters, UsePipes, ValidationPipe } from '@nestjs/common';
import {
  ConnectedSocket,
  MessageBody,
  OnGatewayDisconnect,
  SubscribeMessage,
  WebSocketGateway,
  WebSocketServer,
} from '@nestjs/websockets';
import type { Server, Socket } from 'socket.io';
import { JoinBookDto } from './join-book.dto.js';
import { CommentsWsFilter } from './comments-ws.filter.js';

@UseFilters(CommentsWsFilter)
@WebSocketGateway({
  namespace: 'comments',
  cors: { origin: ['http://localhost:5173'] },
})
export class CommentsGateway implements OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  @SubscribeMessage('join')
  @UsePipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
  join(@MessageBody() body: JoinBookDto, @ConnectedSocket() client: Socket) {
    client.join(body.bookId);
    return { joined: body.bookId };
  }

  announce(bookId: string, comment: { id: string; text: string }) {
    this.server.to(bookId).emit('comment', comment);
  }

  handleDisconnect(client: Socket) {
    client.data.userId = undefined;
  }
}
```

## مثال 2 — المتحكم بعد الحفظ

```ts
@Post(':id/comments')
create(@Param('id') bookId: string, @Body() body: CreateCommentDto) {
  const saved = this.comments.create(bookId, body);
  this.gateway.announce(bookId, { id: saved.id, text: saved.text });
  return saved;
}
```

`CommentsModule` يصدّر البوابة أو يسجّل المتحكم معها حتى يُحقن `CommentsGateway` في اتجاه واحد.

## مثال 3 — عميل الصفحة

```ts
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000/comments');
socket.emit('join', { bookId }, (ack: { joined: string }) => {
  console.log(ack.joined);
});
socket.on('comment', (comment: { id: string; text: string }) => {
  console.log(comment.id);
});
```

عند مغادرة الصفحة: `socket.emit('leave', { bookId })` ثم `socket.disconnect()`.

## ماذا تتأكد منه

- عميل `WebSocket` الخام لا يكمل المصافحة.
- `POST` وحده يحفظ. `join` لا يكتب صفًا.
- قارئ لم ينضم لا يستلم `comment`.
- جسم `{ bookId, userId }` يرفضه الأنبوب.
