# تمارين الفصل

## سهل

أي حافلة لأمر الإلغاء، وأي حافلة لقراءة اللوحة، وأي حافلة بعد `save`؟

<details>
<summary>الحل</summary>

`CommandBus.execute` للإلغاء، `QueryBus.execute` للقراءة، `EventBus.publish` للحدث بعد الحفظ.

</details>

## متوسط

معالج إنشاء كتاب يعيد `books.create` بلا حدث، و `GET` يعيد نفس الأعمدة. ماذا تحذف؟

<details>
<summary>الحل</summary>

الأمر والمعالج ونداء `CommandBus`. تبقى `BooksService.create`. لا حزمة على هذا المسار.

</details>

## متقدم

الإلغاء يحفظ الجذر، وتحديث الولاء والبريد يجب ألا يعيدا الخصم إن فشلا. أين كل قطعة، وماذا لا تضع في `rethrowUnhandled`؟

<details>
<summary>الحل</summary>

القاعدة في `CancelOrderHandler` مع المستودع. بعد `save` حدث `OrderCancelled`. معالجان للولاء والبريد. اترك `rethrowUnhandled` على `false` حتى لا يفشل مسار الأمر بعد الحفظ بسبب أثر، وراقب `UnhandledExceptionBus` لإعادة المحاولة.

</details>
