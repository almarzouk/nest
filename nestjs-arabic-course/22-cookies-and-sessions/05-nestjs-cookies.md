# Cookies في NestJS

## الهدف من هذا الدرس

قراءة الكعكة وكتابتها بواجهة Nest 12.1 التي تعمل على Express و Fastify: `@Cookies()` و `setCookie`.

## المشكلة التي يحلها

`req.cookies` بعد `cookie-parser` يربط المتحكم بـ Express. `reply.setCookie` يربطه بـ Fastify. نقل المشروع بين المنصتين يكسر الدخول. مقال قديم ينسخ `maxAge` بالمللي ثانية فيطوّل العمر.

## الفكرة ببساطة

`@Cookies("name")` يقرأ قيمة من رأس `Cookie`. الكتابة عبر `HttpAdapterHost.httpAdapter.setCookie` و `clearCookie`، مع `@Res({ passthrough: true })` حتى تبقى الاستجابة من قيمة الدالة لا من `res` اليدوي. التوقيع خيار على `NestFactory.create` ثم `@SignedCookies()`.

## التعريف التقني

من صفحة Cookies. المزخرف من `@nestjs/common`. كعكة غائبة ترجع `undefined`. أول تكرار للاسم يفوز. القيمة تُفك ترميزها. الأنابيب على المعامل نوعها `custom`، فـ `ValidationPipe` العام يتخطاها ما لم تُفعَّل `validateCustomDecorators`.

```ts
@Get("theme")
findTheme(@Cookies("library_theme") theme?: string) {
  return { theme: theme ?? "light" };
}
```

الكتابة:

```ts
constructor(private readonly adapterHost: HttpAdapterHost) {}

@Put("theme")
setTheme(
  @Body("theme") theme: string,
  @Res({ passthrough: true }) res: unknown,
) {
  this.adapterHost.httpAdapter.setCookie(res, "library_theme", theme, {
    sameSite: "lax",
    maxAge: 60 * 60 * 24 * 30,
  });
  return { theme };
}
```

`passthrough: true` يترك إرسال الجسم للإطار. `maxAge` ثوانٍ. `httpOnly` افتراضيًا `false` في هذه الواجهة. لكعكة السمة قد تبقيها مقروءة للسكربت. لا تستخدم هذا المثال لكعكة الجلسة: تلك تكتبها `SignInService`.

التوقيع:

```ts
const app = await NestFactory.create(AppModule, {
  cookies: { secret: process.env.COOKIE_SECRET },
});
```

ثم `signed: true` عند الكتابة و `@SignedCookies("uid")` عند القراءة. توقيع تالف أو سر خاطئ يرجع `undefined`. سر `undefined` لا يفشل الإقلاع، والفشل يظهر عند أول كعكة موقّعة، لذلك تحققه في مخطط البيئة. التوقيع يمنع العبث بالقيمة ولا يخفي النص. لا توقّع كلمة مرور داخل الكعكة.

`clearCookie` يصفّر العمر. مرّر `path` و `domain` نفسيهما المستخدمين عند الضبط، والافتراضي `path: "/"`.

`cookie-parser` و `@fastify/cookie` ما زالا يعملان إن كان المشروع عليهما. الواجهة المدمجة لا تلغيها، والصفحة تشرح من يفوز عند اجتماع الاثنين. مشروع جديد يكتفي بالمدمج.

## Mental Model

```text
@Cookies("library_theme")     قراءة
setCookie(..., maxAge ثوانٍ)  كتابة
@SignedCookies                قراءة بعد HMAC
SignInService                 كعكة الهوية، لا هذا المثال
```

## أبسط مثال

```ts
import { Controller, Cookies, Get } from "@nestjs/common";

@Controller("preferences")
export class PreferencesController {
  @Get()
  all(@Cookies() cookies: Record<string, string>) {
    return { theme: cookies.library_theme ?? "light" };
  }
}
```

## شرح المثال

إرجاع كل `cookies` إلى العميل قد يسرّب أسماء كعكات لا تخص العقد، ومنها إن وُجدت قيمة لا يجب أن تعود في JSON. المثال الأسلم يختار `library_theme` فقط. لا تسرد الكعكات في استجابة الإنتاج.

`@Cookies()` بلا اسم يعيد الكائن. مع اسم يعيد النص أو `undefined`.

## ماذا يحدث داخليًا؟

Nest يحلّل رأس `Cookie` حين يطلب معامل ذلك، مرة لكل طلب. `setCookie` يضيف `Set-Cookie` عبر `appendHeader` على المحوّل، فيعمل المسار على Express و Fastify. الاسم غير الصالح أو `sameSite: "none"` بلا `secure: true` يرمي `TypeError` فيصبح `500`. تحقق من الاسم قبل الكتابة إن جاء من مستخدم.

التوقيع بصيغة `cookie-signature`: بادئة `s:` ثم القيمة ثم HMAC. كعكات Express الموقّعة بالسر نفسه تبقى مقروءة. القيمة ليست مشفّرة.

## مثال عملي

`PUT /preferences/theme` بجسم `{ "theme": "dark" }` يضبط الكعكة لثلاثين يومًا ويرجع `{ "theme": "dark" }`. `GET /preferences/theme` من غير كعكة يرجع `light`. `DELETE` يستدعي `clearCookie(res, "library_theme")`. جلسة Ada لا تُمس لأن الاسم مختلف. سر التوقيع إن استخدمته لاحقًا لكعكة غير هوية يأتي من `ConfigService` لا من `"my-secret"` في المقال.

## أخطاء شائعة

- `@Res()` بلا `passthrough` ثم صمت الاستجابة لأن الإطار لم يرسل قيمة الدالة.
- `maxAge` بالمللي ثانية.
- إرجاع خريطة كل الكعكات.
- كتابة `__Host-sid` يدويًا بجانب `SignInService`.
- سر توقيع فارغ لا يوقفه مخطط البيئة.

## متى نستخدمه؟

لكعكة تفضيل أو قيمة غير دليل دخول، بصفات تناسب حساسيتها.

## متى لا نستخدمه؟

لإعادة بناء جلسة الدخول. الحزمة تملك تلك الكعكة. ولا لتخزين `password` حتى لو وُقّعت.

## مقارنة

| `@Cookies` | `req.cookies` بعد cookie-parser |
| --- | --- |
| من `@nestjs/common` | Express غالبًا |
| يعمل على المحوّلات | يربط المنصة |
| التوقيع عبر خيار `cookies` | عبر سر الوسيط |

## تمارين

### تمرين سهل

ماذا يرجع `@Cookies("library_theme")` إن لم تُرسل الكعكة؟

<details>
<summary>الحل</summary>

`undefined`. المسار يقدر أن يعيد الافتراضي `light`.

</details>

### تمرين متوسط

لماذا `passthrough: true` مع `setCookie`؟

<details>
<summary>الحل</summary>

حتى يضبط الرأس ويبقى إرسال جسم الدالة على الإطار. من غير الخيار قد تتولى `res` الاستجابة وتسكت القيمة الراجعة.

</details>

### تمرين متقدم

وقّعت كعكة `uid` والسر غير موجود في البيئة. متى يفشل التطبيق؟

<details>
<summary>الحل</summary>

قد يقلع، ثم يرمي عند أول `setCookie` موقّع أو عند `@SignedCookies` فيصبح `500`. افحص السر في `validationSchema` عند الإقلاع.

</details>

## أسئلة مراجعة

- من أين يُستورد `@Cookies()`؟
- ما وحدة `maxAge`؟
- لماذا لا نكتب كعكة الجلسة هنا؟

## الخلاصة

واجهة Nest تقرأ وتكتب الكعكة على المنصتين. التفضيل مثال مناسب. الهوية تبقى على `SignInService`. التوقيع سلامة لا سرية، والسر من الإعداد.

## ماذا بعد؟

[Session في NestJS](./06-nestjs-session.md) وسيط الجلسة في التوثيق مقابل جلسة الحزمة.

## المصادر الرسمية

- NestJS — Cookies: https://docs.nestjs.com/http/cookies
