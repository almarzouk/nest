# مرحلة Middleware

## الهدف من هذا الدرس

تحديد ماذا يجري في مرحلة Middleware: `app.use` أولًا، ثم ربط الوحدات بترتيب `@Global()` ثم الجذر ثم البعد عن الجذر.

## المشكلة التي يحلها

معرف تتبع يُضاف في Module بعيدة قد يُسجَّل بعد أن وحدة أخرى قرأت الرأس ووجدته فارغًا. ترتيب Middleware هو ترتيب Express: من رُبط أولًا يعمل أولًا، مع قاعدة إضافية بين الوحدات. خلط هذا مع Guard يضع كودًا قبل اختيار المسار في مكان يتوقع معالجًا.

## الفكرة ببساطة

Middleware دالة أو صنف يرى الطلب وينادي `next()` ليكمل السلسلة. العام عبر `app.use()` يعمل قبل أي Middleware ربَطتها Module. ثم وحدات `@Global()`، ثم `AppModule`، ثم الوحدات الأخرى الأقرب إلى الجذر فالأبعد. داخل الوحدة الترتيب هو ترتيب الربط.

كتابة `configure` و `NestMiddleware` فصل Middleware التالي. هنا موضع المرحلة فقط.

## التعريف التقني

التوثيق: أولًا المربوط عامًا مثل `app.use()`، ثم المربوط بالوحدة والمطابق للمسار. التشغيل تسلسلي كـ Express. بين الوحدات: `@Global()` أولًا، ثم الجذر، ثم الباقي مرتبًا حسب المسافة من الجذر في رسم الاستيراد.

المطابقة بالمسار تعني أن Middleware وحدة قد لا تعمل على كل URL. هذا ما زال قبل Guards و Interceptors و Pipes وقبل المعالج.

استثناء يُرمى من Middleware تصل إليه طبقة الاستثناءات، وينطبق Exception Filter العام فقط. مرشح Controller ومرشح المسار لا يريان هذا الاستثناء لأن المعالج لم يُختر.

عدم مناداة `next()` وعدم إنهاء الرد يعلق الطلب. إنهاء الرد من Middleware من غير `next()` يوقف بقية الدورة، بما فيها المعالج. ذلك سلوك السلسلة، لا اختصارًا داخل Nest بعد اختيار المسار.

## Mental Model

```text
1 app.use(requestId)
2 Middleware من كل Module عليها @Global()
3 Middleware مربوطة في AppModule
4 Middleware الوحدات الأبعد، الأقرب إلى الجذر قبل الأبعد
5 ثم Guards
```

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import type { NextFunction, Request, Response } from 'express';
import { AppModule } from './app.module.js';

const app = await NestFactory.create(AppModule);

app.use((req: Request, _res: Response, next: NextFunction) => {
  req.headers['x-request-id'] = req.headers['x-request-id'] ?? 'generated';
  next();
});

await app.listen(3000);
```

## شرح المثال

- هذا البند 2.1. أي Middleware وحدة سترى الرأس إن أكملت `next()`.
- `import type` هنا لأن `Request` نوع Express لا رمز حقن.
- المعالج لاحقًا يقرأ الرأس بـ `@Headers('x-request-id')` إن احتاج. المرحلة لا تعرف اسم الدالة.
- الرمز الذي يولَّد هنا مشترك في الشكل لكل الطلبات التي تدخل التطبيق، لا لمسار الإعارة فقط.

## ماذا يحدث داخليًا؟

بعد `app.use` تمشي Nest على Middleware الوحدات المطابقة للمسار. وحدة `AuditModule` إن كانت `@Global()` تسبق ربطًا على `AppModule` حتى لو استورد الجذر ميزات أخرى أولًا في الملف. ميزة `LoansModule` المستوردة من الجذر أقرب من وحدة يستوردها `LoansModule` نفسها، فالأقرب يعمل قبل الأبعد.

داخل الوحدة الواحدة الترتيب هو ترتيب إضافتك لها، كسلسلة Express. لا تعكس Nest هذا الجزء.

عند `next()` ينتقل التحكم إلى عنصر Middleware التالي ثم إلى Guards. رمي خطأ قبل `next()` يغادر المرحلة نحو المرشح العام.

## مثال عملي

```text
@Global() AuditModule     -> stamp
AppModule                 -> accessLog
LoansModule               -> loansMark
InventoryModule           -> يستوردها LoansModule -> inventoryMark
```

طلب `GET /loans` المطابق لكل الروابط يمر: `app.use`، ثم `stamp`، ثم `accessLog`، ثم `loansMark`، ثم `inventoryMark` لأن المخزون أبعد عن الجذر. بعدها تبدأ Guards. `inventoryMark` لا يرى نتيجة Pipe لأن Pipes لم تبدأ، ولا يغير جسمًا لم يُتحقق منه بعد.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| توقع Guard قبل `app.use` | العام في Middleware يسبق Guards كلها |
| ربط التتبع على وحدة بعيدة وقراءته في وحدة `@Global()` | القارئ يعمل أولًا والحقل ما زال فارغًا |
| رمي خطأ وتوقع مرشح على `@Patch` | المرشح العام فقط |
| إنهاء الرد من غير قصد مع `next()` أيضًا | السلسلة قد تكتب مرتين؛ اختر الإنهاء أو `next()` |

## متى نستخدمه؟

لشيء يخص الطلب قبل معرفة المعالج: معرف تتبع، ختم زمني، أو رفض مبكر على مسار عريض لا يحتاج Decorator المسار. الربط العام بـ `app.use` عندما يجب أن يسبق كل الوحدات.

## متى لا نستخدمه؟

لصلاحية عضو على `PATCH /loans/:id`. ذلك Guard بعد اختيار المسار. لتحويل `id` إلى عدد. ذلك Pipe. لا تضع قاعدة الإعارة هنا؛ الخدمة لم تُستدعَ بعد والمعالج لم يبدأ.

## مقارنة

| الربط | متى يعمل |
| --- | --- |
| `app.use` | قبل Middleware كل الوحدات |
| وحدة `@Global()` | بعد العام وقبل الجذر |
| `AppModule` | بعد `@Global()` وقبل الوحدات الأبعد |
| وحدة مستوردة بعمق أكبر | بعد الوحدة التي استوردتها |
| Guard | بعد انتهاء مرحلة Middleware |

## تمارين

### تمرين سهل

ماذا ينادي Middleware ليصل الطلب إلى Guards؟

<details><summary>الحل</summary>

`next()`. إنهاء الرد من غير `next()` يوقف بقية الدورة.

</details>

### تمرين متوسط

رتّب: `loansMark` على `LoansModule`، `stamp` على وحدة `@Global()`، `requestId` في `app.use`، `accessLog` على الجذر.

<details><summary>الحل</summary>

`requestId`، ثم `stamp`، ثم `accessLog`، ثم `loansMark`.

</details>

### تمرين متقدم

لماذا مرشح مكتوب على `LoansController` لا يصيغ استثناء `stamp`؟

<details><summary>الحل</summary>

`stamp` Middleware وقد تكون على وحدة عامة قبل اختيار Controller. استثناء هذه المرحلة يراه Exception Filter العام فقط.

</details>

## أسئلة مراجعة

- ما البندان الرئيسيان داخل مرحلة Middleware؟
- كيف تُرتَّب الوحدات بعد `app.use`؟
- أي مرشح يرى استثناء المرحلة؟

## الخلاصة

Middleware تبدأ بـ `app.use` ثم بربط الوحدات: `@Global()` ثم الجذر ثم الأبعد حسب رسم الاستيراد، وتسلسل داخل الوحدة كـ Express. بعدها Guards. استثناء المرحلة للمرشح العام.

## ماذا بعد؟

المرحلة 3: مرحلة Guards.

## المصادر الرسمية

- NestJS — Request lifecycle (Middleware): https://docs.nestjs.com/faq/request-lifecycle
