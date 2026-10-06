# EventEmitter و @nestjs/event-emitter

## الهدف من هذا الدرس

بث `book.published` من خدمة واستقباله في صنف مزوّد عبر الحزمة الرسمية.

## المشكلة التي يحلها

`EventEmitter` من Node يعمل، لكنه لا يربط المستمعين بحقن Nest. صنف البريد لن يستلم `MailProducer` بسهولة إن بنيت الملتقط بـ `new`. الحزمة تضع `EventEmitter2` في الحاوية وتكتشف `@OnEvent`.

## الفكرة ببساطة

`EventEmitterModule.forRoot()` مرة. المعلن يحقن `EventEmitter2` وينادي `emit` أو `emitAsync`. المستمع `@OnEvent('book.published')` على مزوّد مسجّل. الاسم سلسلة مشتركة في ثابت حتى لا ينحرف حرف.

## التعريف التقني

```bash
npm i @nestjs/event-emitter
```

```ts
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [EventEmitterModule.forRoot()],
})
export class AppModule {}
```

الاسم الثابت `bookPublished = 'book.published'` والحمولة `{ bookId: string }`.

المعلن:

```ts
import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class BooksService {
  constructor(private readonly events: EventEmitter2) {}

  async publish(bookId: string) {
    await this.prisma.book.update({ where: { id: bookId }, data: { status: 'PUBLISHED' } });
    this.events.emit(bookPublished, { bookId });
  }
}
```

`emit` لا ينتظر المستمع `async`. `await emitAsync` ينتظر إدراج الطابور. لا تنتظر SMTP داخل المستمع.

المستمع:

```ts
import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';

@Injectable()
export class PublishedMailListener {
  constructor(private readonly mail: MailProducer) {}

  @OnEvent(bookPublished)
  async handle(payload: { bookId: string }) {
    await this.mail.enqueuePublished(payload.bookId);
  }
}
```

الصنف في `providers` وإلا فلن يُشترك.

## Mental Model

```text
forRoot
  BooksService.emit(book.published)
    PublishedMailListener.handle
```

## أبسط مثال

```ts
this.events.emit(bookPublished, { bookId });
```

بعد `update` مباشرة. لا `emit` في `catch` لفشل الحفظ.

## شرح المثال

الحمولة كائن `{ bookId }` لا سلسلة عارية. الافتراض بلا `wildcard`. لا تفعّل `book.*` إلا لتشخيص، حتى لا يلتقط المستمع `book.deleted`.

## ماذا يحدث داخليًا؟

`forRoot` يسجّل `EventEmitter2` ويربط `@OnEvent` عند الإقلاع. `emit` لا ينتظر وعد المستمع، وقد يبقى رفضًا غير معالج. المتجر يستعمل `await emitAsync` حتى يُرى فشل `add`، بينما فشل البريد لاحقًا مسؤولية `attempts`.

## مثال عملي

`BooksModule` تسجّل `PublishedMailListener` وتستورد `MailModule` الذي يصدّر `MailProducer`. لا خدمة SMTP. حذف المستمع من `providers` يحفظ الكتاب ولا يدرج بريدًا.

## أخطاء شائعة

- `@OnEvent` على صنف غير مسجّل.
- `emit` قبل `update`.
- تفعيل `wildcard` ونسيان أن `book.*` يلتقط `book.deleted` أيضًا.
- انتظار إرسال SMTP داخل المستمع.

## متى نستخدمه؟

لإعلانات الحالة داخل التطبيق بعد أن يصبح `forRoot` في الجذر.

## متى لا نستخدمه؟

بين خدمتين على شبكتين. لا يوجد وسيط هنا. ولا تستبدل `forRoot` بـ `new EventEmitter()` في الخدمة فيضيع الحقن.

## مقارنة

| الاستدعاء | انتظار المستمع `async` |
| --- | --- |
| `emit` | لا |
| `await emitAsync` | نعم |
| نداء `mail.send` المباشر | نعم، وهو الاقتران الذي نتركه |

## تمارين

### تمرين سهل

أي زينة تسجّل المستمع، ومن أي حزمة؟

<details>
<summary>الحل</summary>
`@OnEvent` من `@nestjs/event-emitter`، بعد `EventEmitterModule.forRoot()`.
</details>

### تمرين متوسط

لماذا الثابت `bookPublished` أفضل من كتابة السلسلة مرتين؟

<details>
<summary>الحل</summary>
حتى لا يعلن المعلن `book.published` ويستمع الآخر لـ `book.pubished`. الطرفان يستوردان الثابت.
</details>

### تمرين متقدم

متى تختار `emitAsync` لمستمع يدرج الطابور؟

<details>
<summary>الحل</summary>
عندما تريد أن يفشل مسار النشر أو يُسجَّل إن رمى `add`، لا أن يضيع الوعد. الإرسال البطيء نفسه يبقى خارج `await` الطويل عند مزوّد البريد.
</details>

## أسئلة مراجعة

- ماذا يُحقن في المعلن؟
- أين يُذكر المستمع حتى يعمل؟
- ماذا تفعل `wildcard` إن فُعّلت بلا انتباه؟

## الخلاصة

`forRoot` ثم `emit` بثابت الاسم وحمولة `{ bookId }`، و `@OnEvent` على مزوّد يدرج الطابور. انتظر `emitAsync` فقط لما يجب أن يراه الطلب.

## ماذا بعد؟

ما الذي صار فضفاضًا في الاقتران، وما الذي لم يُحذف من الاعتماد.

## المصادر الرسمية

- NestJS — Events: https://docs.nestjs.com/techniques/events
