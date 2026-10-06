# Raw Body

## الهدف من هذا الدرس

الإبقاء على بايتات طلب الويب هوك كما وصلت في رف، عبر خيار `rawBody` في Nest، حتى ينجح HMAC.

## المشكلة التي يحلها

`@Body()` يعطي كائنًا. التوقيع حُسب على النص `{"id":"evt_1"}` من غير مسافات. إعادة التسلسل تنتج نصًا آخر فيفشل التوقيع الصحيح، أو يمر توقيع بنيتموه أنتم على النص الجديد فيصبح الفحص بلا معنى.

## الفكرة ببساطة

فعّلوا الاحتفاظ بالجسم الخام عند إنشاء التطبيق. في المتحكم اطلبوا `RawBodyRequest` واقرأوا `rawBody` وهو `Buffer`. حلّلوا JSON فقط بعد نجاح التوقيع. الخيار يعمل مع محلّل الجسم المدمج؛ لا تطفئوا المحلّل بـ `bodyParser: false` ثم تتوقعون أن يملأ Nest الحقل.

## التعريف التقني

صفحة Raw body: السبب الشائع هو تحقق توقيع الويب هوك. على Express:

```ts
const app = await NestFactory.create<NestExpressApplication>(AppModule, { rawBody: true });
```

على Fastify، المحول الوسيط الثاني والخيار `rawBody: true` الثالث، ثم `listen` مع `'0.0.0.0'` إن كان النشر خارج الجهاز كما في درس Fastify. نوع الطلب `RawBodyRequest<Request>` أو `RawBodyRequest<FastifyRequest>` وفيه `req.rawBody`. التحذير في الصفحة: الميزة تتطلب المحلّل العام المدمج. `useBodyParser` يحترم الخيار إن غيّرتم المحلّل. حد الحجم يُرفع من هناك إن كان الحدث أكبر من الافتراضي (100kb على Express).

## Mental Model

```text
بايتات على السلك
  → rawBody Buffer  (للتوقيع)
  → JSON.parse      (بعد التطابق فقط)
```

## أبسط مثال

```ts
import { Controller, Post, type RawBodyRequest, Req } from '@nestjs/common';
import type { Request } from 'express';

@Controller('webhooks/billing')
export class RawBillingController {
  @Post()
  receive(@Req() req: RawBodyRequest<Request>) {
    return { bytes: req.rawBody?.length ?? 0 };
  }
}
```

## شرح المثال

- `rawBody` قد يكون `undefined` إن نسيتم `{ rawBody: true }` أو عطّلتم المحلّل.
- لا توقّعوا `req.body` وهو كائن.
- أعيدوا 400 إن غاب الـ Buffer بدل حساب HMAC على سلسلة فارغة تنجح بالخطأ مع سر فارغ.

## ماذا يحدث داخليًا؟

1. المحول يقرأ التيار.
2. مع الخيار، يحتفظ بنسخة Buffer ويمرر النسخة إلى JSON parser ليملأ `body`.
3. المتحكم يرى الاثنين.
4. التوقيع يستخدم Buffer فقط.
5. `JSON.parse(req.rawBody.toString('utf8'))` بعد التطابق يعطي الحدث. إن فشل التحليل فالجسم ليس JSON رغم توقيع صحيح: 400.

## مثال عملي

```ts
import { BadRequestException } from '@nestjs/common';
import { signaturesMatch, sign } from './sign.js';

export function eventFromRaw(secret: string, header: string | undefined, raw: Buffer | undefined) {
  if (!raw || !header) throw new BadRequestException('missing signature');
  if (!signaturesMatch(sign(secret, raw), header)) throw new BadRequestException('bad signature');
  return JSON.parse(raw.toString('utf8')) as { id: string; type: string };
}
```

هذا ترتيب رف الوارد. الصادر عند الشريك يجب أن يوثَّق بنفس القاعدة: وقّعوا الـ Buffer الذي ترسلونه في `fetch` لا كائنًا تعيدون بناءه لاحقًا.

## أخطاء شائعة

- `bodyParser: false` ثم انتظار `rawBody`. الصفحة تقول إن الميزة تعمل مع المحلّل المدمج فقط.
- نسيان المعامل الثالث على Fastify ووضع `rawBody` مكان المحول.
- قراءة الجسم كـ stream في وسيط آخر قبل Nest فيُستهلك التيار.
- تسجيل `rawBody` كاملًا في السجل فيظهر بيانات فوترة.

## متى نستخدمه؟

على تطبيق يستقبل ويب هوك موقّعًا. فعّلوا الخيار مرة في `bootstrap`.

## متى لا نستخدمه؟

لا تحتاجونه لمسارات JSON العادية بلا توقيع على البايتات. الإبقاء العام رخيص عادة، لكن لا تطبعوا النسخة.

## مقارنة

| المدخل | صالح للتوقيع؟ |
| --- | --- |
| `req.rawBody` | نعم |
| `req.body` كائن | لا |
| `JSON.stringify(req.body)` | لا كبديل عن الأصل |

## تمارين

### تمرين سهل

أين يُفعَّل الخيار: على المتحكم أم في `NestFactory.create`؟

<details>
<summary>الحل</summary>

في `create` عبر `{ rawBody: true }`، مع نوع Express أو كخيار ثالث مع Fastify.

</details>

### تمرين متوسط

التوقيع ينجح في اختبار وحدة على Buffer ويفشل في e2e. ماذا تفحصون أولًا؟

<details>
<summary>الحل</summary>

أن `bootstrap` يمرر `rawBody: true` وأن الاختبار الآلي لا يعيد تسلسل الجسم قبل الإرسال. قارنوا طول `req.rawBody` مع الجسم الذي وقّعتموه في الاختبار.

</details>

### تمرين متقدم

حدث أكبر من حد Express الافتراضي. أين ترفعون الحد من غير إطفاء المحلّل؟

<details>
<summary>الحل</summary>

`app.useBodyParser('json', { limit: '...' })` كما في صفحة Raw body. الدالة تحترم `rawBody`. لا `bodyParser: false`.

</details>

## أسئلة مراجعة

- ما نوع `rawBody`؟
- لماذا يفشل HMAC بعد إعادة التسلسل؟
- ما شرط الصفحة بالنسبة لـ `bodyParser: false`؟
- أين يختلف موضع الخيار بين Express و Fastify؟

## الخلاصة

`rawBody: true` يبقي `Buffer` بجانب JSON. التوقيع على الـ Buffer، والتحليل بعده. إطفاء المحلّل المدمج يلغي الميزة التي توثقها الصفحة.

## ماذا بعد؟

التوقيع قد يفشل مؤقتًا أو ينجح بعد ما انقطع الرد. الدرس التالي كيف يعيد المزود وكيف تعيد رف الإرسال.

## المصادر الرسمية

- NestJS — Raw body: https://docs.nestjs.com/faq/raw-body
