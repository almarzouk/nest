# ترتيب التنفيذ

## الهدف من هذا الدرس

حفظ ترتيب Middleware حين يجتمع `app.use` مع عدة وحدات وعدة دوال داخل `apply`.

## المشكلة التي يحلها

معرّف الطلب الذي يُولَّد في وحدة داخلية يصل متأخرًا عن مسجّل عام قرأ الترويسة وهي فارغة. الترتيب يغيّر ما يراه كل طرف.

## الفكرة ببساطة

عام المنصة أولًا، ثم Middleware الوحدات. داخل الوحدة: ترتيب `apply`. بين الوحدات: الوحدات المعلّمة بـ `@Global()` ثم الوحدة الجذر ثم البقية حسب بعدها عن الجذر في شجرة `imports`.

## التعريف التقني

صفحة Request lifecycle تفصّل Middleware هكذا:

1. Globally bound middleware، مثل `app.use()`.
2. Module bound middleware المطابق للمسار.

وتزيد: Middleware وحدات `@Global()` أولًا، ثم Middleware الوحدة الجذر، ثم Middleware الوحدات الأخرى مرتبة بحسب بعدها عن الجذر في رسم الاستيراد.

داخل قائمة واحدة، الترتيب هو ترتيب الربط، كما في Express. `apply(a, b)` يشغّل `a` ثم `b`. استدعاء `apply` ثانٍ في `configure` يأتي بعد الأول.

هذه الطبقة كلها قبل Guards. استثناء هنا لا يمر على فلتر المسار.

## Mental Model

```text
app.use(requestId)                 1
@Global() AuditModule  auditLog    2
AppModule              shelfStamp  3
LoansModule            traceLoan   4
ثم Guards
```

المسافة عن الجذر: وحدة يستوردها `AppModule` مباشرة أقرب من وحدة تستوردها تلك الوحدة. الأبعد يعمل لاحقًا بين Middleware الوحدات.

## أبسط مثال

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';
import { requestId } from './request-id.middleware.js';

const app = await NestFactory.create(AppModule);
app.use(requestId);
await app.listen(3000);
```

```ts
consumer.apply(traceLoan, shelfStamp).forRoutes(LoansController);
```

على `GET /loans`: `requestId` ثم `traceLoan` ثم `shelfStamp`، وبعدها Guard إن وُجد. هذا يفترض أن `traceLoan` و `shelfStamp` في الوحدة نفسها وأن `requestId` هو `app.use`.

## شرح المثال

- `requestId` في `main.ts` لذا يسبق كل `configure`.
- `traceLoan` قبل `shelfStamp` لأنهما وسيطا `apply` بهذا الترتيب.
- نقل `shelfStamp` إلى استدعاء `apply` أعلى في الملف يعكس ترتيبهما.
- مسار لا يطابق `forRoutes` يتخطى سلسلة الوحدة ويبقى على `app.use` فقط.

## ماذا يحدث داخليًا؟

الإقلاع يجمع السلاسل. عند الطلب تمشي المنصة سلسلة `app.use`، ثم تسأل Nest عن Middleware الوحدات المطابقة وتمررها بالترتيب المحسوب من `@Global()` والجذر والمسافة.

مثال مسافة: `AppModule` تستورد `CirculationModule`، و `CirculationModule` تستورد `LoansModule`. Middleware مسجّل في `CirculationModule` يسبق Middleware `LoansModule` لأن الأول أقرب إلى الجذر. Middleware في وحدة `@Global()` يسبق الاثنين، لأن القاعدة تضع العام أولًا.

إن أردت أن يرى المسجّل ترويسة وضعها غيره، ضع كاتب الترويسة في مرحلة أسبق: `app.use` أو وحدة أقرب إلى الجذر.

بعد آخر `next()` يبدأ أول Guard عام. Middleware لا ينتظر نتيجة المعالج. قياس المدة الكاملة للطلب يحتاج Interceptor حول المعالج، لأن العودة من الخدمة تحدث بعد هذه الطبقة.

## مثال عملي

بوابة تضع `x-request-id` أحيانًا. `requestId` في `app.use` يقرأها أو يولّدها قبل أي وحدة. `AuditModule` العامة تكتب سطرًا يرى المعرّف جاهزًا. `LoansModule` تضيف `x-shelf` على مساراتها فقط. توليد المعرّف داخل `LoansModule` يجعل سطر التدقيق العام يُطبع والمعرّف ما زال فارغًا.

## أخطاء شائعة

- اعتبار ترتيب الملفات في المجلد ترتيب التنفيذ. الترتيب من `app.use` ثم رسم الوحدات ثم `apply`.
- وضع مولّد المعرّف في أبعد وحدة وتوقع أن المسجّل العام يراه.
- نسيان أن `@Global()` يقدّم Middleware تلك الوحدة على الجذر. تعليم وحدة عالمية يغيّر الترتيب.
- انتظار شكل الرد النهائي داخل Middleware. الجسم يُكتب في المعالج أو في Interceptor على طريق العودة.

## متى نستخدمه؟

كلما أضفت Middleware ثانيًا. ارسم السطر: من يكتب الترويسة، ومن يقرأها.

## متى لا نستخدمه؟

لا تستخدم ترتيب Middleware لتعويض Guard أو Pipe. الترتيب داخل الطبقة لا ينقل العمل إلى طبقة أخرى.

## مقارنة

| المصدر | متى يعمل |
| --- | --- |
| `app.use` | أول Middleware |
| وحدة `@Global()` | أول Middleware الوحدات |
| `AppModule.configure` | بعد العام |
| وحدة أبعد في `imports` | بعد الأقرب إلى الجذر |
| `apply(a, b)` | `a` ثم `b` |
| Guard | بعد كل ما سبق |

## تمارين

### تمرين سهل

أيهما يسبق: `app.use(requestId)` أم `configure` في `LoansModule`؟

<details>
<summary>الحل</summary>

`app.use(requestId)`. Middleware المنصة العام قبل Middleware الوحدات.

</details>

### تمرين متوسط

`AuditModule` معلّمة بـ `@Global()` وفيها `auditLog`. `AppModule` فيها `shelfStamp`. أيهما يعمل أولًا على طلب يطابق الاثنين؟

<details>
<summary>الحل</summary>

`auditLog`. Middleware الوحدات العامة قبل Middleware الوحدة الجذر.

</details>

### تمرين متقدم

`AppModule` تستورد `DeskModule`، و `DeskModule` تستورد `LoansModule`. كل واحدة تربط دالة على المسار نفسه. رتب الدالتين مع `app.use` الموجود في `main.ts`.

<details>
<summary>الحل</summary>

`app.use` أولًا، ثم دالة `DeskModule` لأنها أقرب إلى الجذر، ثم دالة `LoansModule`. داخل كل وحدة يبقى ترتيب `apply` المحلي.

</details>

## أسئلة مراجعة

- ما أول مجموعة Middleware في الدورة؟
- كيف ترتّب الصفحة وحدات `@Global()` والجذر والبقية؟
- ماذا يعني ترتيب الوسائط داخل `apply`؟
- هل يستطيع Middleware قراءة JSON الذي سيكتبه المعالج؟

## الخلاصة

`app.use` ثم وحدات عامة ثم الجذر ثم الأبعد. داخل السلسلة ترتيب الكتابة في `apply`. ارسم هذا قبل أن تعتمد ترويسة يكتبها طرف ويقرأها طرف آخر.

## ماذا بعد؟

الدرس التالي يقرر متى هذه الطبقة هي الأداة، ومتى ينتقل العمل إلى Guard أو Pipe أو Interceptor.

## المصادر الرسمية

- [Request lifecycle](https://docs.nestjs.com/faq/request-lifecycle)
- [Middleware](https://docs.nestjs.com/middleware)
