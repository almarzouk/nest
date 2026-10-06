# Multi-stage Build

## الهدف من هذا الدرس

فصل مرحلة التجميع عن مرحلة التشغيل حتى لا يدخل المترجم ولا `devDependencies` إلى الصورة التي تُشغَّل في الإنتاج.

## المشكلة التي يحلها

الصورة الأحادية بعد `npm ci` و `npm run build` ما زالت تحتوي TypeScript و Nest CLI ومصدر `src`. سطح الهجوم أكبر والصورة أثقل. المراحل تبني في صورة وترميها، وتنسخ `dist` و `node_modules` الإنتاج فقط إلى أساس نظيف.

## الفكرة ببساطة

مرحلة `deps` أو `build` تثبّت وتبني. مرحلة `runner` أساس Node نفسه، `npm ci --omit=dev`، نسخ `dist`، مستخدم `node`، و `CMD ["node", "dist/main.js"]`. لا نسخة من `src` إلا إذا احتجت ملفًا وقت التشغيل مثل `schema.prisma` لأمر هجرة منفصل، فانسحبه صراحة.

## التعريف التقني

```dockerfile
FROM node:22.14-bookworm-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22.14-bookworm-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

`--omit=dev` يثبّت ما يحتاجه التشغيل. إن كان `prisma` عميلًا مولَّدًا في `node_modules/.prisma` أثناء البناء، انسخه أيضًا أو أعد `prisma generate` في مرحلة التشغيل باعتماد الإنتاج الذي تحتاجه. لا تفترض أن `dist` وحده يكفي إن كان العميل خارج `dist`.

المرحلة الأولى لا تُشحن. `docker build` الناتج المرحلة الأخيرة افتراضيًا.

نفس حد الإصدار: الوسم 22.14 يلبي 22.12، أو استعمل وسم 20 لا يقل عن 20.19. لا تخلط أساس بناء 22 مع تشغيل 18.

`.dockerignore` ما زال يمنع `.env`. الأسرار التي تحتاجها `npm ci` لمستودع خاص تأتي من بناء BuildKit السري لا من طبقة `ENV` تبقى في التاريخ.

لا `npm run start:dev` في `runner`. لا تنسخ `node_modules` من `build` إن كانت تتضمن dev، إلا إذا حذفتها بخطوة صريحة. `npm ci --omit=dev` أوضح.

## Mental Model

```text
build:  ci كامل + tsc -> dist
runner: ci بلا dev + dist فقط -> node dist/main.js
```

## أبسط مثال

الملف أعلاه. `docker build -t shop-api .` ثم `docker image history shop-api` لا يظهر مرحلة `build` كطبقات نهائية قابلة للتشغيل مع المصدر. حجم الصورة أصغر من الأحادية التي أبقت CLI.

## شرح المثال

`COPY --from=build` يسحب الناتج عبر المراحل لا من جهازك. لذلك البناء قابل للتكرار على CI. `USER node` بعد النسخ الذي تم كجذر في الطبقات السابقة شائع، بشرط أن `dist` مقروء للجميع. إن فشل القراءة اضبط الملكية قبل التبديل.

## ماذا يحدث داخليًا؟

1. المرحلة الأولى تنتج نظام ملفات فيه `dist`.
2. المرحلة الثانية تبدأ من أساس نظيف بلا ذلك النظام إلا ما نسخته.
3. `npm ci --omit=dev` يكتب اعتماديات التشغيل.
4. الأمر الافتراضي `node`.
5. طبقات المرحلة الأولى تبقى في الكاش المحلي لتسريع البناء التالي ولا تُرفع إن رفعت الناتج فقط.
6. `SIGTERM` يصل إلى `node` بفضل صيغة exec.

## مثال عملي

أضف نسخ مخطط Prisma إن هجرتك داخل الصورة:

```dockerfile
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=build /app/prisma ./prisma
```

الأفضل توليد العميل في مرحلة `runner` بعد `npm ci` إن كانت حزمة `prisma` ضمن dependencies لا devDependencies. اختر طريقًا واحدًا وجربه بحاوية فارغة لا تعتمد على `node_modules` جهازك.

## أخطاء شائعة

- تشغيل مرحلة `build` بالخطأ عبر `--target build` في الإنتاج.
- نسيان `.prisma` فتفشل الإقلاع رغم وجود `dist`.
- `omit=dev` ثم بناء TypeScript في المرحلة الثانية بلا مترجم.
- أساس تشغيل أقدم من حد Node.
- نسخ `.env` في أي مرحلة.

## متى نستخدمه؟

في كل صورة إنتاج بعد أن يفهم الفريق المرحلة الواحدة.

## متى لا نستخدمه؟

صورة أداة تطوير محلية تريد CLI والمصدر. ملف `compose` للتطوير يمكن أن يبني هدف `build` أو يركب المجلد. لا تخلط الهدفين في `CMD` واحد.

## مقارنة

| المرحلة | فيها src | فيها CLI | تُشغَّل في الإنتاج |
| --- | --- | --- | --- |
| build | نعم | نعم | لا |
| runner | لا | لا | نعم |

## تمارين

### تمرين سهل

ماذا يفعل `--omit=dev` في مرحلة التشغيل؟

<details>
<summary>الحل</summary>

يثبّت اعتماديات الإنتاج فقط، بلا TypeScript وأدوات الاختبار إن كانت في devDependencies.

</details>

### تمرين متوسط

لماذا لا تنسخ `node_modules` من مرحلة البناء كما هي؟

<details>
<summary>الحل</summary>

لأنها تتضمن اعتماديات التطوير. مرحلة التشغيل تثبّت مجموعة الإنتاج أو تحذف الزائد صراحة.

</details>

### تمرين متقدم

الإقلاع يرمي أن عميل Prisma غير مولَّد رغم نسخ `dist`. ماذا نقص؟

<details>
<summary>الحل</summary>

ناتج التوليد في `node_modules/.prisma` أو خطوة `prisma generate` في مرحلة التشغيل. `dist` وحده لا يحوي العميل المولَّد إن بقي خارجها.

</details>

## أسئلة مراجعة

- أي مرحلة تحتوي `CMD`؟
- لماذا الأساس يتكرر في المرحلتين؟
- ماذا لا يُنسخ من `build`؟

## الخلاصة

البناء ينتج `dist` ثم يُرمى من صورة التشغيل. التشغيل `npm ci --omit=dev` و `node dist/main.js` على Node حديث، مع عميل Prisma إن كان خارج `dist`.

## ماذا بعد؟

docker-compose: تشغيل هذه الصورة مع Postgres و Redis بأسمائها.

## المصادر الرسمية

- Docker — Multi-stage builds: https://docs.docker.com/build/building/multi-stage/
- NestJS — Deployment: https://docs.nestjs.com/deployment
