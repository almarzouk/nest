# Structured Logging

## الهدف من هذا الدرس

كتابة كل سطر JSON بحقول ثابتة: المستوى، والوقت، والسياق، والرسالة، ومعرفات مثل `bookId` لا نصًا ملتصقًا.

## المشكلة التي يحلها

البحث عن الكتاب 42 في نص `published book 42 after isbn 978...` هش. مجمع السجلات يفلتر على حقل. نص حر يتغير مع صياغة الجملة فيضيع التنبيه. الحقول الثابتة تبقى عندما تعيد صياغة العربية أو الإنجليزية في الرسالة.

## الفكرة ببساطة

`write` يطبع `JSON.stringify` لكائن واحد في السطر. الرسالة للإنسان. المعرفات حقول. لا تضع الكائن كله إن كان فيه سر. المستوى أسماء ثابتة: `log` و `warn` و `error` و `debug` و `fatal`.

## التعريف التقني

```ts
private write(level: string, message: unknown, context?: string, stack?: string, extra?: Record<string, string>) {
  const line = {
    level,
    time: new Date().toISOString(),
    context: context ?? null,
    message: typeof message === 'string' ? message : 'non-string message',
    ...extra,
    stack,
  };
  process.stdout.write(`${JSON.stringify(line)}\n`);
}
```

الخدمة لا تبني JSON بيدها. تستدعي واجهة صغيرة على مسجّلك أو تمرر كائنًا يفهمه غلاف حول `Logger`. إن بقي `Logger` الافتراضي، يمكن تسجيل رسالة قصيرة مع الإبقاء على المعرف في بداية ثابتة `bookId=42` كحل مؤقت. الهدف في الإنتاج حقول.

مثال نداء:

```ts
this.logger.log({ message: 'published', bookId });
```

إن كان تنفيذك يمرر الكائن إلى `write` كما في `message`، فكّكه: إن كان كائنًا فيه `message` فخذ بقية المفاتيح إلى `extra` بعد حذف أي مفتاح اسمه `authorization` أو `token` أو `password`.

قائمة ممنوع دخولها الحقول: كلمة المرور، `ISBN_TOKEN`، ترويسة `Authorization`، جسم بطاقة، ومحتوى ملف الغلاف.

## Mental Model

```text
سطر واحد
  level time context message
  bookId أو isbn أو jobId
بلا أسرار
```

## أبسط مثال

```json
{"level":"log","time":"2026-10-05T06:00:00.000Z","context":"BooksService","message":"published","bookId":"42"}
```

التنبيه في المجمع: `level=error` و `context=IsbnService`. لا بحث عن كلمة «فشل» العربية والإنجليزية معًا.

## شرح المثال

`time` بصيغة ISO حتى يرتب المجمع الأحداث. `bookId` حقل لا جزء من `message`. تغيير الجملة من `published` إلى `book published` لا يكسر لوحة تراقب `bookId`. `stack` يظهر في الخطأ فقط حتى لا تتضخم أسطر `log`.

## ماذا يحدث داخليًا؟

كل نداء يسلسل كائنًا ويكتب سطرًا مع `\n`. المجمع يقرأ stdout سطرًا سطرًا. كسر السطر داخل `message` يفسد التقسيم، لذلك استبدل الأسطر الجديدة أو اترك المكدس في حقل `stack` المهرّب داخل JSON فيبقى سطرًا واحدًا. `JSON.stringify` يهريب `\n` في القيم. لا تطبع ثم تلحق المكدس خارج الكائن كما في المسجّل النصي السابق.

حجم السطر: لا تلحق مصفوفة كل كتب الصفحة. المعرف والعدد يكفي: `{ message: 'list', count: 20 }`.

## مثال عملي

فشل الناشر:

```json
{"level":"error","context":"IsbnService","message":"upstream failed","isbn":"9780000000001","status":502}
```

بلا `headers` وبلا `responseBody`. رمز HTTP الخارجي في `status` يكفي لمعرفة أن الخريطة أنتجت `502` واردًا. المهلة:

```json
{"level":"error","message":"isbn timed out","isbn":"9780000000001","code":"ECONNABORTED"}
```

## أخطاء شائعة

- جملة عربية فيها المعرف الوحيد.
- لصق `JSON.stringify(axiosError)` في `message`.
- عدة أسطر غير مهرّبة لحدث واحد.
- حقل `password` من جسم `POST` «للتصحيح».

## متى نستخدمه؟

عندما تُشحن السجلات إلى مجمع. في التمرين المحلي النص المقروء مقبول، والشكل نفسه يسهّل التعود على الحقول.

## متى لا نستخدمه؟

لتسجيل كل استعلام Prisma مع المعاملات. الضجيج يخفي `error`. فعّل ذلك مؤقتًا على `debug` لا على `log`.

## مقارنة

| الشكل | البحث عن كتاب 42 |
| --- | --- |
| نص `published 42` | هش |
| JSON فيه `bookId` | حقل مساوٍ |
| كائن Axios كامل | تسريب وبحث بلا داعٍ |

## تمارين

### تمرين سهل

لماذا المعرف حقل لا كلمة في الجملة؟

<details>
<summary>الحل</summary>

حتى يبقى الفلتر ثابتًا عندما تتغير صياغة الرسالة.

</details>

### تمرين متوسط

ماذا تضع لفشل ISBN بدل جسم الرد؟

<details>
<summary>الحل</summary>

`isbn` و `code` أو `status` و `message` ثابتة. بلا ترويسات وبلا جسم.

</details>

### تمرين متقدم

لماذا يُهرَّب المكدس داخل JSON لا يُطبع بعده؟

<details>
<summary>الحل</summary>

حتى يبقى الحدث سطرًا واحدًا يقرأه المجمع. سطر ثانٍ يُحسب حدثًا آخر.

</details>

## أسئلة مراجعة

- أي حقول دنيا في كل سطر؟
- أي مفاتيح تُحذف إن وصلت في الكائن؟
- لماذا العدد أفضل من قائمة كاملة؟

## الخلاصة

سطر JSON بحقول ثابتة ومعرفات. الرسالة للإنسان والأسرار لا تدخل السطر. المكدس حقل لا حدث ثانٍ.

## ماذا بعد؟

سطر لكل طلب HTTP: الطريقة والمسار والرمز والمدة، من غير الجسم السري.

## المصادر الرسمية

- NestJS — Logger: https://docs.nestjs.com/techniques/logger
