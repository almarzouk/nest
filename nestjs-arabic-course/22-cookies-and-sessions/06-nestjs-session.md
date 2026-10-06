# Session في NestJS

## الهدف من هذا الدرس

قراءة وسيط الجلسة كما في صفحة Nest، وتمييزه عن جلسة `@nestjs/authentication` التي تحمل هوية المكتبة.

## المشكلة التي يحلها

نسخ `app.use(session({ secret: "my-secret" }))` من مقال يجعل كل زائر يملك سجلًا في الذاكرة، والسر ضعيفًا، والهوية مكررة بجانب `__Host-sid`. على Fastify الواجهة مختلفة: `@fastify/secure-session` لا `express-session`.

## الفكرة ببساطة

المنصة الافتراضية Express: وسيط `express-session` يوقّع كعكة المعرّف ويضع كائنًا على الطلب. `@Session()` يقرأ ذلك الكائن. Fastify: مكوّن `secure-session` ثم `session.get` و `session.set`، أو `@Session()` أيضًا. هذا الوسيط مناسب لسلة أو عداد زيارات في تطبيق يعرض HTML. هوية API الدورة تبقى جلسة الحزمة بمخزن تكتبه أنت.

## التعريف التقني

من صفحة Session. Express:

```bash
npm i express-session
npm i -D @types/express-session
```

```ts
import session from "express-session";

app.use(
  session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
  }),
);
```

الصفحة تقول إن المخزن الافتراضي للذاكرة ليس للإنتاج. `secret` يوقّع كعكة المعرّف، ويمكن أن يكون مصفوفة للتدوير: الأول يوقّع والكل يتحقق. `resave: false` لا يعيد الكتابة إن لم يتغير السجل. `saveUninitialized: false` لا يحفظ جلسة جديدة لم تُعدَّل، وهذا أنسب لدخول حقيقي ويقلل السجلات.

`cookie.secure: true` موصى به مع HTTPS. خلف وكيل اضبط `trust proxy` وإلا فلن تُرسل الكعكة الآمنة.

```ts
@Get("visits")
visits(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
  return { visits: session.visits };
}
```

`@Session()` من `@nestjs/common`. نوع Express من حزمة `express` إن قرأت `req.session` عبر `@Req()`.

Fastify في الصفحة نفسها: `@fastify/secure-session` مع `secret` أطول من `32` حرفًا و `salt`، ثم `session.set("visits", n)`.

لا تضع `passwordHash` على `session`. لا تستخدم هذا الوسيط لـ `POST /posts` في الدورة ما دام `SessionAuth` يثبت المستخدم. جلستان تعني كعكتين وسرين ومخزنين.

إنتاج `express-session` يحتاج مخزنًا مثل `connect-pg-simple` أو ما يعادله، وهذا خارج مثال الصفحة. حزمة المصادقة توثّق واجهة `SessionStore` مباشرة. للهوية اتبع تلك الواجهة على Prisma.

## Mental Model

```text
express-session -> req.session -> عداد أو سلة
@nestjs/authentication -> __Host-sid -> هوية المكتبة
لا تجمع الاثنين على نفس معنى «من المستخدم»
```

## أبسط مثال

```ts
import { Controller, Get, Session } from "@nestjs/common";

@Controller("stats")
export class StatsController {
  @Get("visits")
  visits(@Session() session: { visits?: number }) {
    const visits = (session.visits ?? 0) + 1;
    session.visits = visits;
    return { visits };
  }
}
```

## شرح المثال

العداد تفضيل جهاز لا حساب. ضياع الكعكة يبدأ العد من جديد. هذا صادق للدالة. لو كان `visits` صلاحية نشر لكان كارثة، لأن أي متصفح يحصل على كعكة موقّعة جديدة بعد أن يكتب الوسيط سجلًا. الهوية لا تُستنتج من وجود `req.session` فارغ.

`saveUninitialized: false` قد لا يحفظ الجلسة حتى تُسند خاصية. الإسناد هنا يعدّل السجل فيُحفظ.

## ماذا يحدث داخليًا؟

الوسيط يقرأ كعكة المعرّف، يتحقق من التوقيع، يحمّل السجل من المخزن، ويكتبه في نهاية الطلب إن تغيّر. مخزن الذاكرة يربط السجل بالعملية. إعادة التشغيل تنسى العداد. نسختان خلف موازن لا تشاركان العد. هذا مقبول لتجربة محلية وغير مقبول لحساب دخول.

حزمة المصادقة لا تمر عبر هذا الوسيط لتثبت `@CurrentUser()`. مزوّدها يقرأ كعكتها. تشغيل الوسيط بلا داعٍ يضيف كعكة ثانية على كل زائر إن تركت `saveUninitialized` على سلوك يحفظ الجلسة الفارغة.

## مثال عملي

أبقِ `StatsController` خارج مسار الهوية إن أردت تجربة الصفحة الرسمية. `GET /stats/visits` يزيد العدد. `GET /me` ما زال يعتمد كعكة الحزمة فقط. احذف الوسيط من `main.ts` إن لم يوجد قارئ `@Session()` حتى لا توزّع كعكة إضافية. سر `SESSION_SECRET` من البيئة إن بقي الوسيط، بطول عشوائي، لا `"my-secret"` من المثال التعليمي في الإنتاج.

على Fastify بدّل التسجيل إلى `secure-session` كما في الصفحة، ولا تستدعِ `express-session`.

## أخطاء شائعة

- السر `"my-secret"` في الإنتاج.
- مخزن الذاكرة لهوية المستخدمين.
- جلسة Express موازية لـ `__Host-sid`.
- تخزين الدور على `req.session` ونسيانه عند سحب الصلاحية.
- نسخ إعداد Express إلى Fastify.

## متى نستخدمه؟

لبيانات واجهة صغيرة على تطبيق Express أو Fastify حسب صفحتهما، حين لا تكون حزمة المصادقة هي المخزن.

## متى لا نستخدمه؟

بديلًا عن `SessionStore` في `@nestjs/authentication` لمشروع الدورة. الهوية لها مسار واحد.

## مقارنة

| `express-session` | جلسة الحزمة |
| --- | --- |
| `req.session` و `@Session()` | `@CurrentSession()` |
| مثال الصفحة للوسيط | `__Host-sid` |
| مخزن افتراضي ذاكرة للتجربة | الإنتاج واجهة مخزن |
| عداد الزيارات | حساب المستخدم |

## تمارين

### تمرين سهل

من أين يُستورد `@Session()`؟

<details>
<summary>الحل</summary>

من `@nestjs/common`.

</details>

### تمرين متوسط

لماذا `saveUninitialized: false` مناسب لصفحة الدخول؟

<details>
<summary>الحل</summary>

حتى لا يُنشأ سجل لكل زائر لم يسجّل الدخول ولم تُعدَّل جلسته، فيقل التخزين وتُحترم المتصفحات التي لم توافق على كعكة بعد.

</details>

### تمرين متقدم

المشروع على Fastify ونسخت لقطة `app.use(session(...))`. لماذا لا تُحفظ الزيارة؟

<details>
<summary>الحل</summary>

`express-session` وسيط Express. صفحة Nest على Fastify تسجّل `@fastify/secure-session` عبر `app.register`، والقراءة بـ `session.get` و `session.set`.

</details>

## أسئلة مراجعة

- ماذا يحذّر التوثيق عن المخزن الافتراضي؟
- أي جلسة تثبت مستخدم المكتبة؟
- ماذا يفعل `secret` في الوسيط؟

## الخلاصة

صفحة Session توثّق وسيط المنصة لسجل مثل عداد، وتحذّر من الذاكرة في الإنتاج. هوية الدورة تبقى كعكة الحزمة ومخزنها. لا تُنشئ جلستين لمعنى الدخول نفسه.

## ماذا بعد؟

[أمثلة عملية](./examples.md) لكعكة سمة بجانب جلسة الدخول من غير خلط.

## المصادر الرسمية

- NestJS — Session: https://docs.nestjs.com/http/session
- NestJS — Cookies: https://docs.nestjs.com/http/cookies
- NestJS — Authentication: https://docs.nestjs.com/security/authentication
