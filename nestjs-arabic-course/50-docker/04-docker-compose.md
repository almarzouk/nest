# docker-compose

## الهدف من هذا الدرس

تعريف خدمات `api` و `postgres` و `redis` على شبكة واحدة، مع نشر المنافذ وفحص جاهزية القاعدة قبل اعتبارها قابلة للاتصال.

## المشكلة التي يحلها

تشغيل ثلاثة أوامر Docker يدويًا يخطئ في الشبكة والأسماء. ملف Compose يصف الخدمات والمتغيرات والمنافذ. `depends_on` وحده ينتظر بدء الحاوية لا جاهزية Postgres لقبول اتصال. شرط `service_healthy` مع فحص صحة يقلل فشل الإقلاع المبكر. عنوان القاعدة داخل الملف اسم الخدمة.

## الفكرة ببساطة

`docker compose up --build` يبني `api` ويشغّل الكل. من جهازك تفتح `localhost:3000` لأن المنفذ منشور. من حاوية `api` القاعدة على `postgres:5432` و Redis على `redis:6379`. لا تضع هذه الأسماء في كود Prisma. ضعها في `DATABASE_URL`.

## التعريف التقني

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://app:app@postgres:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7
volumes:
  pgdata:
```

أمر `api` يأتي من `CMD` في Dockerfile: `node dist/main.js`. لا تكتب `command: npm run start:dev` على هذه الخدمة إن كان الملف إنتاجيًا. ملف `compose.dev.yml` منفصل يجوز أن يركّب المصدر ويشغّل المراقبة على الجهاز.

`ports` ينشر للضيف. الخدمات ترى بعضها بلا نشر المنفذ على الجهاز. لا تنشر `5432` على الإنترنت في الإنتاج. النشر المحلي للتطوير.

كلمات `app` هنا للتعلم المحلي لا لصورة تُرفع. في بيئة مشتركة أسرار من ملف غير متتبع أو من مدير المنصة.

## Mental Model

```text
ضيفك -> localhost:3000 -> حاوية api:3000
api -> postgres:5432
api -> redis:6379
pgdata يبقى بعد الإيقاف
```

## أبسط مثال

الملف أعلاه في جذر المستودع. `docker compose up --build`. بعد جاهزية Postgres يبدأ `api`. إن فشل الاتصال فما زالت السلسلة تشير إلى `localhost`. ابحث في المتغير لا في صورة Postgres.

## شرح المثال

`service_healthy` ينتظر نجاح `pg_isready`. من دونها يبدأ Node قبل أن يسمع المنفذ فيفشل `$connect` ويعيد التشغيل إن كانت السياسة كذلك. Redis غالبًا يصبح جاهزًا أسرع، و `service_started` أقل صرامة. أضف فحص `redis-cli ping` إن رأيت سباقًا.

## ماذا يحدث داخليًا؟

1. Compose ينشئ شبكة المشروع.
2. يسجل اسم `postgres` و `redis` و `api`.
3. الحجم `pgdata` يُربط ببيانات القاعدة.
4. الفحص يتكرر حتى النجاح أو استنفاد المحاولات.
5. `api` تُبنى من Dockerfile متعدد المراحل وتستلم البيئة.
6. `compose down` يوقف الحاويات. `down -v` يحذف البيانات. لا تستعمله على حجم فيه ما تحتاجه.

## مثال عملي

أضف خدمة `nats` أو `rabbitmq` بنفس الأسلوب عندما تصل إلى النقل: `RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672` للتطوير فقط. لا تخلط ملف الإنتاج الذي يسحب صورًا من سجل مع ملف التطوير الذي يبني المصدر. سمِّ الملفات صراحة.

## أخطاء شائعة

- `DATABASE_URL` فيها `localhost`.
- `command: npm run start:dev` على صورة أنتجت `dist` فقط فتفشل لأن `src` غير موجود.
- نشر Postgres على `0.0.0.0` في خادم عام.
- نسيان الحجم ثم العجب من ضياع البيانات.
- اعتبار `depends_on` بلا فحص يكفي دائمًا.

## متى نستخدمه؟

تشغيل محلي متكرر و CI يحتاج التطبيق مع القاعدة.

## متى لا نستخدمه؟

عنقود إنتاج كبير قد تديره منصة أخرى. يبقى Dockerfile هو عقد الصورة. Compose للتطوير ولنشر بسيط تفهم حدوده.

## مقارنة

| الحقل | دوره |
| --- | --- |
| `build` | يصنع صورة `api` |
| `image` | يسحب Postgres أو Redis |
| `environment` | عناوين بأسماء الخدمات |
| `healthcheck` | جاهزية لا مجرد بدء |
| `volumes` | بقاء بيانات القاعدة |

## تمارين

### تمرين سهل

من أين يفتح المتصفح التطبيق، ومن أين تفتحه حاوية أخرى؟

<details>
<summary>الحل</summary>

المتصفح عبر المنفذ المنشور `localhost:3000`. حاوية أخرى عبر اسم الخدمة والمنفذ الداخلي، لا عبر منفذ الضيف بالضرورة.

</details>

### تمرين متوسط

لماذا `pg_isready` وليس `depends_on` فقط؟

<details>
<summary>الحل</summary>

بدء الحاوية يسبق قبول Postgres للاتصالات. الفحص ينتظر الجاهزية.

</details>

### تمرين متقدم

لماذا فشل `start:dev` داخل هذه الـ `api` حتى بعد تصحيح الأمر في Dockerfile؟

<details>
<summary>الحل</summary>

صورة التشغيل بلا `src` وبلا CLI. المراقبة للتطوير بتركيب المجلد في ملف Compose منفصل، لا بتغيير أمر الإنتاج.

</details>

## أسئلة مراجعة

- ماذا يفعل `-v` مع `down`؟
- أي اسم في `DATABASE_URL`؟
- ماذا لا يُنشر على الإنترنت؟

## الخلاصة

Compose يضع `api` و `postgres` و `redis` على شبكة بأسماء هي المضيفات، وينتظر جاهزية القاعدة، ويحفظ البيانات في حجم. أمر التطبيق يبقى تشغيل `dist`.

## ماذا بعد؟

NestJS و PostgreSQL و Redis معًا: المتغيرات والإقلاع والإيقاف داخل هذا الملف.

## المصادر الرسمية

- Docker — Compose: https://docs.docker.com/compose/
- NestJS — Deployment: https://docs.nestjs.com/deployment
