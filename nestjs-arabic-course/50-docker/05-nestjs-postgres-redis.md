# NestJS و PostgreSQL و Redis

## الهدف من هذا الدرس

جمع الصورة متعددة المراحل مع Compose بحيث يتصل Nest بـ Postgres و Redis بأسماء الخدمات، ويغلق العملاء عند إيقاف الحاوية.

## المشكلة التي يحلها

ملف Compose صحيح مع كود ما زال يثبت `localhost`، أو صورة تشغّل `start:dev`، أو إيقاف Docker يقطع Node قبل `$disconnect`. هذا الدرس يربط الإعداد والإقلاع والإيقاف بالعناوين داخل الشبكة.

## الفكرة ببساطة

`DATABASE_URL` مضيفه `postgres`. `REDIS_URL` مضيفه `redis`. `ConfigModule` يقرأهما عند الإقلاع. `PrismaService` على `onModuleInit` يتصل وعلى `onModuleDestroy` يقطع. `main.ts` يستدعي `enableShutdownHooks()` حتى يمر `SIGTERM` من Docker إلى الخطافات. الأمر في الصورة `node dist/main.js`.

## التعريف التقني

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks();
  await app.listen(process.env.PORT ?? 3000, '0.0.0.0');
}
await bootstrap();
```

`0.0.0.0` حتى يصل نشر المنفذ `3000:3000` إلى العملية.

```ts
DATABASE_URL: postgresql://app:app@postgres:5432/app
REDIS_URL: redis://redis:6379
```

عميل Redis في مصنع أو في `onModuleInit` يستعمل `REDIS_URL` كما هو. لا تستبدل المضيف في الكود بشرط `if (docker)`.

الهجرة: `npx prisma migrate deploy` خطوة قبل تبديل المرور أو حاوية مهمة تنتهي، لا سباق كل نسخة `api` عند كل بدء. في التطوير يمكن تشغيلها مرة من جهازك نحو منفذ منشور لـ Postgres إن نشرته محليًا، أو مهمة Compose لمرة واحدة.

الصحة لاحقًا تفحص القاعدة. هنا يكفي أن فشل `$connect` يمنع الاستماع لأن الوعد في `onModuleInit` يرفض الإقلاع.

Node في الصورة 22.14 أو أي وسم يحقق 22.12+ أو 20.19+. لا تشغّل الصورة بقاعدة Node أقدم من حد Nest.

## Mental Model

```text
SIGTERM من Docker
  -> enableShutdownHooks
  -> onModuleDestroy: $disconnect و redis.quit
  -> العملية تخرج
الإقلاع: مضيف postgres و redis من البيئة
```

## أبسط مثال

`PrismaService` من فصل دورة الحياة بلا تغيير سوى أن العنوان يأتي من البيئة التي وضعها Compose. أضف عميل ذاكرة:

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  constructor(private readonly config: ConfigService) {}

  async onModuleInit(): Promise<void> {
    const url = this.config.getOrThrow<string>('REDIS_URL');
    await this.connect(url);
  }

  async onModuleDestroy(): Promise<void> {
    await this.quit();
  }

  private async connect(url: string): Promise<void> {
    void url;
  }

  private async quit(): Promise<void> {
    return undefined;
  }
}
```

استبدل `connect` بمكتبة `ioredis` التي تستقبل `url` كاملًا. لا تمرر `host: 'localhost'` بجانب العنوان.

## شرح المثال

الخدمتان Singleton. الإقلاع ينتظر الاتصالين قبل `listen` إن كانا في وحدات تُهيأ قبل الاستماع. الإيقاف من `docker compose stop` يرسل `SIGTERM`، والخطاف يحرر المقابس حتى لا تقتل المهلة العملية في منتصف الكتابة. من غير `enableShutdownHooks` Docker ينتظر ثم يرسل `SIGKILL` والعملاء يُقطعون.

## ماذا يحدث داخليًا؟

1. Compose يمرر البيئة إلى العملية.
2. `ConfigService` يقرأ `DATABASE_URL` بعد `@` حتى المضيف `postgres`.
3. DNS للشبكة يحل الاسم إلى عنوان الحاوية.
4. `$connect` يفتح مجمعًا نحو `5432` داخل شبكة Compose لا عبر منفذ الضيف.
5. عند الإيقاف تُستدعى `onModuleDestroy` بالترتيب الموثق.
6. إن بقي مؤقت بلا `clearInterval` قد لا تخرج العملية قبل `SIGKILL`.

## مثال عملي

تحقق من داخل الحاوية لا من جهازك فقط:

```text
docker compose exec api node -e "console.log(process.env.DATABASE_URL)"
```

يجب أن ترى المضيف `postgres`. ثم أوقف `postgres` وتأكد أن نسخة جديدة من `api` لا تفتح المنفذ وهي تتصل بفراغ. أعد القاعدة و `up` مرة أخرى. `docker compose stop api` ثم راجع أن سجل الإيقاف ظهر من `onModuleDestroy` إن أضفت سطر سجل.

## أخطاء شائعة

- مضيف `localhost` في `DATABASE_URL` داخل Compose.
- `start:dev` في `command`.
- هجرة متوازية من كل نسخة.
- نسيان `enableShutdownHooks` ثم مهلة قتل عشر ثوانٍ.
- نشر `5432` واعتماد التطبيق على `localhost:5432` من داخل الحاوية رغم النشر. النشر للضيف فقط.

## متى نستخدمه؟

كلما شغّلت المتجر بـ Compose مع القاعدة والذاكرة.

## متى لا نستخدمه؟

لا تضع كلمة سر التعلم `app` في خادم عام. ولا تشغّل Redis الناقل و Redis الكاش بلا قصد على نفس الاستخدام إن كان الفصل أوضح لحملك.

## مقارنة

| المتغير | المضيف داخل Compose | على جهازك بلا Docker |
| --- | --- | --- |
| `DATABASE_URL` | `postgres` | `127.0.0.1` |
| `REDIS_URL` | `redis` | `127.0.0.1` |
| أمر التشغيل | `node dist/main.js` | `start:dev` للتطوير |

## تمارين

### تمرين سهل

لماذا `listen` على `0.0.0.0` مع نشر المنفذ؟

<details>
<summary>الحل</summary>

حتى تقبل العملية الاتصالات القادمة عبر واجهة الحاوية التي ينشرها Docker، لا على واجهة داخلية فقط.

</details>

### تمرين متوسط

ماذا يرسل `docker compose stop` وكيف يصل إلى `$disconnect`؟

<details>
<summary>الحل</summary>

`SIGTERM`. مع `enableShutdownHooks` تدخل سلسلة الإيقاف ومنها `onModuleDestroy`.

</details>

### تمرين متقدم

لماذا لا تشغّل `migrate deploy` في `CMD` نفسه مع خادم له عدة نسخ؟

<details>
<summary>الحل</summary>

النسخ تتسابق على الهجرة. نفّذها خطوة واحدة قبل توجيه المرور، ثم ابدأ النسخ على مخطط مستقر.

</details>

## أسئلة مراجعة

- أين يُكتب اسم المضيف؟
- ماذا يغلق `onModuleDestroy` هنا؟
- أي أمر يُمنع كـ `command` للإنتاج؟

## الخلاصة

Nest في Compose يقرأ عناوين `postgres` و `redis` من البيئة، يستمع على `0.0.0.0`، ويغلق العملاء عندما تصل إشارة Docker بفضل خطافات الإيقاف. الصورة تشغّل `dist` بإصدار Node المطلوب.

## ماذا بعد؟

أمثلة الملفات الثلاثة معًا، ثم الإنتاج على الخادم.

## المصادر الرسمية

- Docker — Compose: https://docs.docker.com/compose/
- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
- NestJS — Deployment: https://docs.nestjs.com/deployment
