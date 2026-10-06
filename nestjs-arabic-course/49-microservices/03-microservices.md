# Microservices

## الهدف من هذا الدرس

تعريف خدمة Nest الصغيرة كتطبيق بطبقة نقل غير HTTP، متى تفصلها، وكيف تُنشأ بـ `createMicroservice`.

## المشكلة التي يحلها

تقسيم المجلدات لا يعزل العطل. الخدمة الصغيرة عملية مستقلة تتحدث برسالة. في Nest هي تطبيق يستعمل ناقلًا بدل HTTP أو معه. التوثيق يبنيها بـ `NestFactory.createMicroservice`. تفصل عندما تحتاج نشرًا وحد فشل مستقلين لسياق حدّه واضح في الـ Modular monolith. تفصل قبل ذلك فتنقل الدورة إلى الشبكة.

## الفكرة ببساطة

عملية الكتالوج تبقى HTTP للواجهة. عملية الفوترة تستمع على ناقل. العقد نمط رسالة لا مسارًا داخليًا. كل عملية لها إصدارها. لا استيراد TypeScript بين المستودعين إلا لمكتبة عقد صغيرة إن اخترت مشاركتها. الفشل الجزئي طبيعي: الفوترة قد تكون غائبة والكتالوج يعرض الكتب.

## التعريف التقني

الحزمة `@nestjs/microservices`.

```ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    { transport: Transport.TCP },
  );
  await app.listen();
}
await bootstrap();
```

TCP هو الافتراضي في التوثيق. `listen` هنا يبدأ نقل الخدمة لا منفذ HTTP. التطبيق الهجين يبني HTTP بـ `create` ثم `connectMicroservice` ثم `startAllMicroservices()` ثم `listen` للمنفذ.

المفاهيم السابقة تبقى: حقن، حراس، Pipes، فلاتر، على حدود يدعمها النقل. `@MessagePattern` مكان `@Get` لطلب ينتظر ردًا. `@EventPattern` لحدث بلا رد.

كل عملية `enableShutdownHooks()` حتى تغلق اتصال الناقل عند `SIGTERM`.

لا تشارك قاعدة يكتبها الطرفان. لكل خدمة ملكية بياناتها. النسخة عبر رسالة أو واجهة قراءة. المشاركة تعيد الـ Monolith بتعطّل أوضح.

## Mental Model

```text
متصفح -> HTTP Catalog process
Catalog -> رسالة -> Billing process
كل عملية تنشر وحدها وتسقط وحدها
```

## أبسط مثال

عملية الفوترة بالمقطع أعلاه. المعالج:

```ts
import { Controller } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class InvoiceController {
  @MessagePattern({ cmd: 'invoice.sum' })
  sum(data: number[]): number {
    return data.reduce((total, item) => total + item, 0);
  }
}
```

العميل في درس `ClientProxy` يرسل `{ cmd: 'invoice.sum' }`. النمط جزء من العقد مثل المسار.

## شرح المثال

`createMicroservice` لا يفتح REST. الاختبار بـ `supertest` لا يرى هذا المعالج. الاختبار يستدعي الدالة أو عميل الناقل. `sum` مثال التوثيق على نمط `{ cmd: 'sum' }` مصغّرًا للفوترة. المنطق الحقيقي يبقى خدمة بلا تفاصيل النقل.

## ماذا يحدث داخليًا؟

1. العملية تتصل بالناقل أو تفتح مقبس TCP.
2. رسالة تحمل نمطًا وحمولة.
3. الإطار يطابق النمط إلى دالة المتحكم.
4. الناتج يرجع لمسار الطلب-الرد أو يُهمل في الحدث.
5. عملية ثانية لا تشارك ذاكرة `EventBus` المحلي.
6. إعادة النشر لعملية لا تعيد تشغيل الأخرى.

## مثال عملي

استخرج الفوترة فقط. الكتالوج HTTP هجين يرسل أمر إصدار. الفوترة تملك جداول الفاتورة. الكتالوج يحتفظ `invoiceId`. عند تعطل الفوترة يعرض الكتالوج الكتب ويُظهر تعذر الدفع. لا تنقل البحث والكتالوج معًا «ليكون التقسيم كاملًا».

## أخطاء شائعة

- خدمة لكل جدول.
- قاعدة مشتركة للكتابة.
- TCP بين حاويات بلا اتفاق منفذ في شبكة Docker (العنوان اسم الخدمة لا `localhost` عند العبور بين الحاويات).
- نسيان أن `@MessagePattern` يُIgnore خارج المتحكم.
- استخراج بلا حد وحدة سابق.

## متى نستخدمه؟

نشر مستقل وحد فشل مستقل لسياق جاهز، مع ناقل تشغّله.

## متى لا نستخدمه؟

لتجميل المجلدات أو لمسار CRUD واحد. الـ Modular monolith أرخص.

## مقارنة

| | Modular monolith | Microservice |
| --- | --- | --- |
| `main` | `create` + HTTP | `createMicroservice` أو هجين |
| العقد الداخلي | دالة مصدَّرة | نمط رسالة |
| السقوط | العملية كلها | العملية المعنية |
| البيانات | ملكية منطقية | ملكية عملية |

## تمارين

### تمرين سهل

ماذا يفتح `createMicroservice` مع `Transport.TCP` بدل `NestFactory.create`؟

<details>
<summary>الحل</summary>

مستمع نقل للرسائل، لا تطبيق HTTP وحده. `listen` يبدأ ذلك النقل.

</details>

### تمرين متوسط

لماذا مشاركة جدول الفاتورة كتابةً من الكتالوج تلغي العزل؟

<details>
<summary>الحل</summary>

لأن تعطّل المخطط أو القفل يضرب العمليتين، والملكية لم تنتقل مع العملية.

</details>

### تمرين متقدم

كيف تبقي HTTP للواجهة وتستمع للناقل في نفس العملية أثناء انتقال تدريجي؟

<details>
<summary>الحل</summary>

تطبيق هجين: `create` ثم `connectMicroservice` ثم `startAllMicroservices` ثم `listen`. الهدف المؤقت نقل الفوترة لا إبقاء الهجين إلى الأبد بلا سبب.

</details>

## أسئلة مراجعة

- ما الحزمة؟
- أين يُكتب النمط؟
- ماذا يبقى مشتركًا في الذاكرة؟

## الخلاصة

الخدمة الصغيرة عملية بنقل رسائل وعقد نمط وملكية بيانات. أنشئها بـ `createMicroservice` بعد حدود الـ Modular monolith، لا بدلًا من تلك الحدود.

## ماذا بعد؟

أساليب التواصل: طلب ينتظر ردًا، وحدث يُنشر بلا انتظار.

## المصادر الرسمية

- NestJS — Microservices: https://docs.nestjs.com/microservices/basics
