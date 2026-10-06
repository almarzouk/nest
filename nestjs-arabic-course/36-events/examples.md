# أمثلة عملية

نشر كتاب: الحفظ في الخدمة، والآثار مستمعون.

## مثال 1 — العقد

```ts
export const bookPublished = 'book.published';

export interface BookPublishedPayload {
  bookId: string;
}
```

```ts
EventEmitterModule.forRoot();
```

بلا `wildcard`.

## مثال 2 — المعلن

```ts
await this.prisma.book.update({
  where: { id: bookId },
  data: { status: 'PUBLISHED' },
});
await this.events.emitAsync(bookPublished, { bookId });
```

## مثال 3 — مستمعان مستقلان

```ts
@OnEvent(bookPublished)
async queueMail(payload: BookPublishedPayload) {
  await this.mail.enqueuePublished(payload.bookId);
}

@OnEvent(bookPublished)
async dropListCache(payload: BookPublishedPayload) {
  await this.cache.del('books:published');
  await this.cache.del(`book:${payload.bookId}`);
}
```

كلا الصنفين في `providers`. لا `smtp.send` هنا. لا حذف مفتاح `draft:${bookId}` كنص مسودة.

## ماذا تتأكد منه

- اختبار `publish` يتوقع `emitAsync` فقط.
- اختبار المستمع يتوقع `enqueuePublished('42')`.
- قتل العملية بعد `add` لا يضيع المهمة. قتلها أثناء SMTP المباشر يضيعها، لذلك SMTP ليس في المستمع.
