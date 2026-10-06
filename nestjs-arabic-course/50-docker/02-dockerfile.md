# Dockerfile

## الهدف من هذا الدرس

كتابة صورة تشغّل Nest مبنيًا: اعتماديات إنتاج، ناتج `dist`، مستخدم غير جذر، وأمر `node dist/main.js`.

## المشكلة التي يحلها

نسخ مجلد التطوير كله مع `node_modules` من جهازك يجر أدوات البناء وأسرارك ويختلف بين المعماريات. Dockerfile يصف البناء من مصدر نظيف. الصورة الناتجة تشغّل الناتج فقط. الدرس التالي يفصل مرحلة البناء عن مرحلة التشغيل. هنا شكل الملف والقرارات التي تبقى في الحالتين.

## الفكرة ببساطة

أساس `node:22.14-bookworm-slim` أو صورة 20 تلبي 20.19 على الأقل. مجلد العمل `/app`. انسخ بيان الحزم أولًا حتى تُخزن طبقة `npm ci`. ثم انسخ المصدر. ابنِ. شغّل `node dist/main.js`. لا `CMD npm run start:dev`.

## التعريف التقني

صورة أحادية المرحلة للتعلم، قبل التصغير:

```dockerfile
FROM node:22.14-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
ENV NODE_ENV=production
USER node
CMD ["node", "dist/main.js"]
```

`npm ci` يحترم القفل. `npm install` داخل الصورة قد يغيّر الإصدارات. `.dockerignore` يستبعد `node_modules` و `dist` و `.env` و `.git` حتى لا تُنسخ من جهازك فوق تثبيت نظيف.

`USER node` موجود في صورة Node الرسمية. التشغيل كجذر يجعل ثغرة في العملية ثغرة على صلاحيات عالية. إن احتجت كتابة ملفات، امنح مجلدًا محددًا لا الجذر.

`EXPOSE 3000` توثيق. النشر الفعلي في Compose عبر `ports`.

Prisma: ولّد العميل قبل `nest build` إن كان السكربت لا يفعل ذلك (`npx prisma generate`). ملفات `schema.prisma` يجب أن تكون داخل سياق البناء. لا تشغّل migrations تلقائيًا بلا قرار: غالبًا خطوة نشر منفصلة لا كل بدء حاوية تتسابق عليها النسخ.

الاستماع: التطبيق على `0.0.0.0` إن كان `listen` مربوطًا بواجهة محلية فقط فلن يراك نشر المنفذ. `app.listen(port, '0.0.0.0')` عندما تقيد الواجهة.

## Mental Model

```text
package*.json -> npm ci -> مصدر -> build -> node dist/main.js
.dockerignore يمنع .env و node_modules المحلية
USER node
```

## أبسط مثال

الملف أعلاه مع `.dockerignore`:

```text
node_modules
dist
.git
.env
coverage
```

`docker build -t shop-api .` ثم `docker run --rm -p 3000:3000 -e DATABASE_URL=... shop-api`. المتغير عند التشغيل لا في `COPY .env`.

## شرح المثال

طبقة الحزم تُعاد فقط عندما يتغير القفل. طبقة المصدر تتغير مع الكود. `NODE_ENV=production` بعد التثبيت في هذه الصورة الأحادية قد يزيل حاجة devDependencies قبل البناء إن ضبطته مبكرًا. لذلك المرحلة الواحدة حرجة الترتيب: ثبّت كل شيء، ابنِ، ثم صورة التشغيل في الدرس التالي تحذف أدوات البناء. لا تضع `NODE_ENV=production` قبل `npm ci` إن كان البناء يحتاج أدوات التطوير، إلا إذا فصلت المراحل.

## ماذا يحدث داخليًا؟

1. الأساس يجلب Node بالإصدار المحدد.
2. `npm ci` يكتب `node_modules` في طبقة.
3. `nest build` أو `npm run build` يكتب `dist`.
4. عند التشغيل الأمر الافتراضي هو شكل exec في `CMD` فيستلم الإشارات. شكل shell قد لا يمرر `SIGTERM` إلى Node فلا تعمل خطافات الإيقاف.
5. `USER node` يحد الصلاحيات.
6. تغيير سطر مصدر يلغي كاش الطبقات بعده لا طبقات الحزم إن بقي الترتيب.

## مثال عملي

`package.json` فيه `"start:prod": "node dist/main.js"` ويمكن `CMD ["npm", "run", "start:prod"]`. استدعاء `node` مباشرة أوضح وأقل عملية وسطى. لا تستخدم `start:dev` ولا `start:debug`. أضف `dumb-init` فقط إن وجدت أن إشارات صورتك لا تصل إلى Node. صورة Node الرسمية مع `CMD` بصيغة JSON تمرر الإشارات إلى العملية الأولى.

## أخطاء شائعة

- `CMD npm run start:dev`.
- نسخ `.env`.
- نسيان `.dockerignore` فيُرسل سياق ضخم.
- `NODE_ENV=production` قبل تثبيت أدوات البناء في مرحلة واحدة فيفشل `nest build`.
- التشغيل كجذر بلا سبب.
- `latest` بدل وسم Node الذي يطابق 22.12+ أو 20.19+.

## متى نستخدمه؟

لكل نشر حاوية للتطبيق. ملف واحد في جذر المستودع.

## متى لا نستخدمه؟

لا تبنِ الصورة لتشغيل اختبار وحدة. الاختبار على الجهاز أو خطوة CI من غير أن يصبح Dockerfile أداة الاختبار الوحيدة.

## مقارنة

| الأمر | أين |
| --- | --- |
| `npm run start:dev` | جهازك |
| `npm run build` | أثناء بناء الصورة |
| `node dist/main.js` | `CMD` الإنتاج |
| `npm ci` | طبقة الحزم |

## تمارين

### تمرين سهل

لماذا `package.json` يُنسخ قبل المصدر كله؟

<details>
<summary>الحل</summary>

حتى تُخزن طبقة `npm ci` ولا تُعاد عند كل تغيير سطر في `src`.

</details>

### تمرين متوسط

لماذا صيغة `CMD ["node", "dist/main.js"]` أنسب من shell لخطافات الإيقاف؟

<details>
<summary>الحل</summary>

صيغة exec تجعل Node العملية الأولى فتستلم `SIGTERM`. غلاف shell قد يبتلع الإشارة فلا يعمل `enableShutdownHooks`.

</details>

### تمرين متقدم

أين تضع `prisma generate` نسبة إلى `npm run build`؟

<details>
<summary>الحل</summary>

قبل البناء حتى توجد الأنواع والعميل أثناء التجميع، داخل الصورة لا على جهازك فقط. الهجرة خطوة نشر منفصلة عن بدء كل نسخة.

</details>

## أسئلة مراجعة

- ماذا يستبعد `.dockerignore`؟
- أي مستخدم يشغّل العملية؟
- أي أمر يُمنع في الإنتاج؟

## الخلاصة

Dockerfile يثبّت بالقفل، يبني `dist`، ويشغّل `node dist/main.js` كمستخدم `node` بصيغة exec. لا أسرار في السياق، ولا `start:dev`، وإصدار Node يطابق حد Nest.

## ماذا بعد؟

Multi-stage Build: صورة تشغيل بلا مترجم ولا اعتماديات تطوير.

## المصادر الرسمية

- Docker — Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- NestJS — Deployment: https://docs.nestjs.com/deployment
