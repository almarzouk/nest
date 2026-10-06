# REPL

## الهدف من هذا الدرس

فتح حاوية رف في الطرفية واستدعاء مزود singleton، مثل قراءة مشروع، عبر `repl()` من `@nestjs/core`.

## المشكلة التي يحلها

كتابة مسار HTTP مؤقت لتجربة دالة. أو تشغيل `publish` من الطرفية على قاعدة الإنتاج لأن التابع ظاهر. أو `get()` على مزود بنطاق الطلب فيرجع الحاوية شكلًا بلا سياق مستأجر.

## الفكرة ببساطة

ملف `repl.ts` بجانب `main.ts` يستدعي `await repl(AppModule)`. التشغيل `npm run start -- --entryFile repl`. الدوال الأصلية `get` و `methods` و `debug` و `resolve` و `select` تعمل في الجلسة. `get` للمزود العادي. `resolve` لنطاق عابر أو نطاق طلب، ومعه سياق. REPL أداة تطوير لا مدخل إنتاج.

## التعريف التقني

```ts
import { repl } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  await repl(AppModule);
}
await bootstrap();
```

الأمر: `npm run start -- --entryFile repl`. المراقبة: `npm run start -- --watch --entryFile repl`، والتاريخ يُحفظ بـ `replServer.setupHistory('.nestjs_repl_history', ...)`. `get(token)` أو `$` يجلب نسخة مسجّلة ويرمي إن غابت. `methods(ProjectsService)` يسرد التوابع العامة. `debug()` يطبع الوحدات. `debug(ProjectsModule)` يضيّق. `select` ينتقل في شجرة الوحدات. `help()` القائمة، و `$.help` توقيع الدالة. `repl()` يعيد خادم REPL في Node ويمكن تمرير خياراته وسيطًا ثانيًا.

## Mental Model

```text
repl(AppModule) → حاوية بلا listen
get(خدمة singleton) → استدعاء تابع
resolve + سياق → نطاق الطلب
```

## أبسط مثال

بعد الإقلاع:

```text
get(ProjectsService)
methods(ProjectsService)
debug(ProjectsModule)
```

`ProjectsService` يجب أن يكون قابلًا للاستدعاء في الجلسة (الاستيراد في `repl.ts` أو مرجع من وحدة محمّلة). النتيجة كائن الخدمة لا JSON مسار.

## شرح المثال

- لا منفذ HTTP. الحراس لا يلفّون التابع الذي تستدعونه بأيديكم.
- مرّروا `organizationId` وسيطًا صريحًا. لا تضعوا معرف مكتبة تجريبي داخل الخدمة من أجل الجلسة.
- `methods` يعرض العام فقط. تابع خاص لن يظهر ولن يُستدعى من الخارج عن قصد.

## ماذا يحدث داخليًا؟

1. `repl` يبني التطبيق كما `NestFactory` ويهيئ الوحدات ويطبع `REPL initialized`.
2. الدوال تُحقن في نطاق الجلسة العام.
3. `get` يبحث في الوحدات عن الرمز. الوضع غير الصارم يبدأ من الجذر.
4. مزود `Scope.REQUEST` بلا سياق طلب لا يُؤخذ بـ `get` العادي. `resolve(token, contextId)` للنطاق العابر أو الطلب.
5. `--watch` يعيد بناء الجلسة ويمسح التاريخ ما لم يُضبط `setupHistory`.

## مثال عملي

قراءة مسودة دار النيل من تطوير محلي:

```text
svc = get(ProjectsService)
await svc.findOne('org_nile', 'prj_season')
```

`findOne` هنا يفترض التوقيع الذي يفرض المنظمة وسيطًا. إن أعادت الدالة صفًا بلا هذا الوسيط فالجلسة تكشف التصميم الضعيف. لا تستدعوا `publish` ولا `Mailer` على بيانات حقيقية من REPL جهاز المطوّر.

## أخطاء شائعة

- `--entryFile` قبل `--` في أمر npm فلا يصل إلى Nest.
- تجربة خصم Stripe أو نشر لأن التابع نجح في الجلسة.
- `get` على خدمة طلب ثم قراءة `tenantId` فارغ.
- فتح REPL على `DATABASE_URL` الإنتاج.

## متى نستخدمه؟

لاستكشاف الرسم البياني وتجربة تابع خالص على قاعدة تطوير.

## متى لا نستخدمه؟

بدل اختبار Vitest. الجلسة لا تثبت العقد ولا تُشغَّل في CI. ولا بدل `createApplicationContext` لمهمة مجدولة على الخادم.

## مقارنة

| الأداة | من يستدعيها |
| --- | --- |
| REPL | مطوّر على الطرفية |
| Vitest | CI وأمر `test` |
| مسار HTTP | العميل والحارس |

## تمارين

### تمرين سهل

أي ملف وأي أمر يفتحان الجلسة؟

<details>
<summary>الحل</summary>

`repl.ts` فيه `await repl(AppModule)`، والأمر `npm run start -- --entryFile repl`.

</details>

### تمرين متوسط

ما الفرق بين `get` و `resolve`؟

<details>
<summary>الحل</summary>

`get` نسخة مسجّلة عادية. `resolve` لنطاق عابر أو نطاق طلب ويأخذ `contextId`. مزود المستأجر لكل طلب لا يُجلب بـ `get` وحده.

</details>

### تمرين متقدم

لماذا نجاح `publish` داخل REPL لا يعني أن الحارس سمح به؟

<details>
<summary>الحل</summary>

الجلسة تستدعي التابع على الكائن مباشرة. الأنابيب والحراس والمرشحات المرتبطة بالمسار لا تعمل. المنظمة يجب أن تأتي وسيطًا منكم، والاختبار هو الذي يثبت المسار.

</details>

## أسئلة مراجعة

- من أي حزمة `repl`؟
- كيف تُحفظ أوامر الجلسة مع `--watch`؟
- ماذا يطبع `debug()`؟
- أي قاعدة بيانات تُفتح؟

## الخلاصة

REPL يحمّل `AppModule` في الطرفية ويستدعي المزودات بـ `get` و `resolve`. هو عدسة تطوير. النشر والفوترة يبقى لهما مسار واختبار لا سطر جلسة.

## ماذا بعد؟

حفظ الملف اليوم يعيد الترجمة. الدرس التالي يفصل إعادة التشغيل عن التحميل الساخن.

## المصادر الرسمية

- NestJS — REPL: https://docs.nestjs.com/recipes/repl
