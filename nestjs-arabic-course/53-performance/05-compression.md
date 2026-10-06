# Compression

## الهدف من هذا الدرس

تقليل حجم جسم HTTP الخارج من متجر رف عندما يكون النص كبيرًا، ومعرفة أن التوثيق يفضّل نقل الضغط عن عملية Node في المواقع عالية الحركة.

## المشكلة التي يحلها

`GET /books` يعيد عشرين كتابًا بجسم JSON من عشرات الكيلوبايتات. على شبكة الجوال، النقل أغلى من توليد JSON. الضغط يجعل الجسم أصغر. تشغيله داخل كل عملية Nest يستهلك CPU على الحلقة نفسها التي تخدم البيع.

## الفكرة ببساطة

العميل يرسل `Accept-Encoding`. الخادم يضغط الجسم بخوارزمية يقبلها العميل ويكتب `Content-Encoding`. المتصفح أو عميل HTTP يفك الضغط قبل أن يرى تطبيقك JSON. الصور المضغوطة أصلًا لا تربح شيئًا يذكر، وتكلّف CPU.

## التعريف التقني

على Express، حزمة `compression` تُسجَّل بـ `app.use(compression())` بعد إنشاء التطبيق، وتدعم gzip و deflate و Brotli حسب `Accept-Encoding`. على Fastify، الحزمة `@fastify/compress` تُسجَّل بـ `app.register` على `NestFastifyApplication`، وتفضّل افتراضيًا zstd ثم Brotli ثم gzip ثم deflate على إصدارات Node التي تدعم ذلك. جودة Brotli الافتراضية في الإضافة 4 لا 11، لأن الجودة الأعلى أبطأ. صفحة Compression تقول بوضوح: للمواقع عالية الحركة في الإنتاج، انقل الضغط إلى وكيل عكسي مثل Nginx ولا تستخدم وسيط الضغط داخل التطبيق عندها.

## Mental Model

```text
جسم JSON 80KB
→ gzip داخل Node أو داخل Nginx
→ على السلك ~15KB
→ العميل يفك إلى نفس الكائن
```

المعنى لا يتغير. الذي يتغير وقت CPU ومكانه.

## أبسط مثال

```ts
import compression from 'compression';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(compression());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

## شرح المثال

- هذا مسار Express الافتراضي. `compression()` وسيط عام قبل المسارات.
- الأجسام الصغيرة قد تُترك كما هي حسب عتبة الحزمة، وهذا مطلوب حتى لا يكلف الضغط أكثر من النقل.
- لا تضف الوسائط على Fastify بهذا الاستيراد. هناك الإضافة الخاصة، وتسجيلها يحتاج نوع `NestFastifyApplication`.

## ماذا يحدث داخليًا؟

1. الطلب يصل ومعه `Accept-Encoding: gzip, br`.
2. الـ Handler يبني كائن JavaScript.
3. Nest يسلسله إلى JSON.
4. وسيط الضغط يضغط البايتات إن تجاوزت العتبة ونوع المحتوى مناسب.
5. الاستجابة تحمل `Content-Encoding` و `Vary: Accept-Encoding` حتى لا تخزن طبقة وسيطة نسخة مضغوطة لمن لا يقبلها.

## مثال عملي

Fastify مع جودة Brotli صريحة، كما في التوثيق:

```ts
import { constants } from 'node:zlib';
import compression from '@fastify/compress';
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, type NestFastifyApplication } from '@nestjs/platform-fastify';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
  await app.register(compression, {
    brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } },
  });
  await app.listen(process.env.PORT ?? 3000, '0.0.0.0');
}
bootstrap();
```

على رف في الإنتاج أمام Nginx، اترك التطبيق بلا هذا التسجيل واضبط `gzip` أو `brotli` على الوكيل للاستجابات `application/json`. قياس CPU العملية قبل وبعد هو الذي يبرر المكان.

## أخطاء شائعة

- ضغط الصور `image/jpeg` وملفات PDF الجاهزة.
- تفعيل الضغط في Node وNginx معًا فيضغط الجسم مرتين أو تتعارض الترويسات.
- الحكم على الربح بحجم الملف على القرص لا بقياس p95 و CPU تحت الحمل.
- نسيان `Vary` عند تخزين الاستجابة في CDN، فيصل gzip إلى عميل طلب الهوية بلا ضغط.

## متى نستخدمه؟

JSON كبير يتكرر على شبكات بطيئة، وCPU العملية فيه متسع، أو — وهو الأفضل تحت الحمل — الوكيل أمام التطبيق يتولى الضغط.

## متى لا نستخدمه؟

عندما تكون الأجسام أصغر من عتبة الحزمة، أو المحتوى مضغوطًا، أو القياس يُظهر أن CPU الضغط يرفع زمن الاستجابة أكثر مما يوفره النقل. وعندها أزل الوسيط من التطبيق.

## مقارنة

| المكان | متى |
| --- | --- |
| `compression` على Express | تطوير أو حركة متوسطة بعد قياس |
| `@fastify/compress` | نفس القرار على محول Fastify |
| Nginx / الوكيل | إنتاج عالي الحركة، كما توصي الصفحة |

## تمارين

### تمرين سهل

عميل يرسل `Accept-Encoding: gzip` ويستلم JSON في الكود. من فك الضغط؟

<details>
<summary>الحل</summary>

عميل HTTP (المتصفح أو المكتبة) يفك `Content-Encoding` قبل أن يسلم النص لتطبيق الواجهة. كود Nest ما زال يعيد كائنًا، والضغط يحدث على البايتات بعد التسلسل.

</details>

### تمرين متوسط

لماذا لا نبقي `app.use(compression())` على رف وقد أصبح خلف Nginx يضغط أصلًا؟

<details>
<summary>الحل</summary>

الصفحة الرسمية تطلب عدم استخدام وسيط الضغط داخل التطبيق عندما ينقل الإنتاج هذه المهمة إلى الوكيل. الضغط مرتين يهدر CPU وقد يفسد الترويسات. اتركه في طبقة واحدة.

</details>

### تمرين متقدم

قياس أظهر أن Brotli بجودة 11 خفض الحجم 8٪ إضافية ورفع CPU العملية إلى الضعف. ماذا تغيّر؟

<details>
<summary>الحل</summary>

أخفض الجودة (الافتراضي في `@fastify/compress` هو 4) أو انقل الضغط إلى الوكيل حيث يمكن ضبطه بمعزل عن حلقة الطلب. الحجم ليس الهدف الوحيد.

</details>

## أسئلة مراجعة

- أي ترويسة يعلن بها العميل الخوارزميات المقبولة؟
- ماذا توصي صفحة Compression للموقع عالي الحركة؟
- لماذا نوع التطبيق `NestFastifyApplication` شرط لتسجيل الإضافة؟
- أي أنواع ملفات لا تستفيد من الضغط؟

## الخلاصة

الضغط يصغّر النص على السلك ويكلف CPU. في Express الحزمة `compression`، وفي Fastify `@fastify/compress`. تحت الحمل الثقيل مكانه الوكيل لا حلقة Nest.

## ماذا بعد؟

حتى مع جسم أصغر، عملية Node واحدة تستخدم نواة واحدة للحساب. الدرس التالي عن تشغيل أكثر من عملية.

## المصادر الرسمية

- NestJS — Compression: https://docs.nestjs.com/http/compression
