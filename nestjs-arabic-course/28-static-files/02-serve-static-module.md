# ServeStaticModule

## الهدف من هذا الدرس

تسجيل `@nestjs/serve-static` حتى يُخدم مجلد `public` تحت بادئة `/assets` من غير التقاط مسارات الـ API.

## المشكلة التي يحلها

نسخ `app.use(express.static(...))` في `main.ts` يعمل، لكنه يختفي عن شجرة الوحدات ويصعب استثناء `/api`. الوحدة الديناميكية تعلن الجذر والبادئة والاستثناءات حيث تُراجع مع بقية `imports`.

## الفكرة ببساطة

`ServeStaticModule.forRoot` يأخذ كائنًا أو أكثر. `rootPath` مجلد القرص. `serveRoot` بادئة العنوان. `exclude` مسارات لا تُلتقط (على Express). `serveStaticOptions` تُمرر إلى `express.static` أو ما يقابله على Fastify.

## التعريف التقني

من واجهة الحزمة:

| الخاصية | المعنى |
| --- | --- |
| `rootPath` | مجلد الملفات. الافتراض `"client"` |
| `serveRoot` | بادئة العنوان. الافتراض `""` أي جذر الموقع |
| `renderPath` | المسار الذي تُرسم عليه الواجهة. الافتراض على Express `{*any}` |
| `exclude` | مسارات مستبعدة. غير مدعوم على Fastify |
| `useGlobalPrefix` | إن `true` تُسبق الملفات بالبادئة العامة من `setGlobalPrefix` |
| `serveStaticOptions` | خيارات الطبقة السفلى: `index` و `fallthrough` و `maxAge` و `dotfiles` |

`exclude` في مثال Nest الرسمي يشبه `/api/{*test}` لأن `path-to-regexp` الحديث يكتب البدل بهذا الشكل. نمط قديم مثل `/api*` لا يطابق ما تتوقعه. على Fastify استخدم `renderPath` كتعبير منتظم بدل `exclude`.

`dotfiles` الافتراضي `ignore` حتى لا يُرسل `.env` إن سقط داخل المجلد.

## Mental Model

```text
GET /assets/logo.svg
  serveRoot /assets يطابق
  الملف public/logo.svg يُرسل
GET /books/42
  exclude أو بادئة مختلفة
  المتحكم يكمل
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'node:path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '..', 'public'),
      serveRoot: '/assets',
      serveStaticOptions: {
        index: false,
        fallthrough: false,
        maxAge: '1h',
      },
    }),
  ],
})
export class AppModule {}
```

في قالب ESM، `__dirname` قد تحتاج بناءه من `import.meta.url` إن لم يوفره إعداد TypeScript. المسار النهائي يجب أن يشير إلى `public` نفسه في التطوير وبعد `nest build`.

## شرح المثال

`serveRoot: '/assets'` يمنع الوحدة من الجلوس على `/`. شعار المتجر: `GET /assets/logo.svg`. `index: false` حتى لا يُرسل `index.html` لطلب مجلد. `fallthrough: false` يجعل غياب الملف تحت `/assets` ينتهي هنا بخطأ ثابت بدل ما ينزلق إلى متحكم فيُفسَّر كمعرّف كتاب. `maxAge` يضع `Cache-Control` للأصول التي تتغير مع الإصدار.

## ماذا يحدث داخليًا؟

`forRoot` يسجل الخيارات كمزوّد. عند الإقلاع يركب المحمّل وسيطًا على المحول HTTP. الطلب يُقارن بـ `serveRoot` ثم بمسار نسبي تحت `rootPath`. `..` في العنوان لا يُفترض أن يخرج من الجذر لأن طبقة `express.static` تحصر الجذر. مع ذلك لا تبنِ `rootPath` من إدخال مستخدم. `exclude` يُترجم إلى اختبار مسار. إن طابق الطلب نمط الاستثناء، الوسيط لا يخدم ملفًا ويترك السلسلة تكمل نحو المتحكمات.

## مثال عملي

متجر يضع API تحت `/api` وأصولًا تحت `/assets`، مع استثناء صريح حتى لو تغيّرت البادئة لاحقًا:

```ts
ServeStaticModule.forRoot({
  rootPath: join(process.cwd(), 'public'),
  serveRoot: '/assets',
  exclude: ['/api/{*path}'],
  serveStaticOptions: { dotfiles: 'ignore', index: false },
});
```

`process.cwd()` واضح في التشغيل المحلي. في Docker ثبّت مجلد العمل في الصورة حتى لا يشير `cwd` إلى مسار فارغ.

## أخطاء شائعة

- `rootPath` على `var/covers`.
- `exclude: ['/api*']` ثم استغراب أن `/api/books` ما زال يُلتقط.
- توقع عمل `exclude` على Fastify.
- `fallthrough: true` على جذر الموقع، فيعود `index.html` لمسار API غير موجود.

## متى نستخدمه؟

لأصول تُنشر مع الخادم وتُطلب بالاسم: صور الواجهة، ملفات `public`، بناء ثابت تحت بادئة.

## متى لا نستخدمه؟

لتنزيل مشروط بصلاحية. ذلك `StreamableFile`. ولتكبير الواجهة عبر CDN، الخادم قد لا يخدم الأصول أصلًا.

## مقارنة

| الخيار | أثره إن نُسي |
| --- | --- |
| `serveRoot` | الملفات تجلس على `/` وتزاحم API |
| `exclude` | مسار API غير المطابق قد يبتلعه البدل |
| `fallthrough: false` | الغياب ينزلق إلى طبقة أخرى |
| `dotfiles: 'ignore'` | ملف نقطة قد يُرسل إن غيّرت الخيار إلى `allow` |

## تمارين

### تمرين سهل

أي خاصية تجعل الشعار تحت `/assets` لا تحت `/logo.svg`؟

<details>
<summary>الحل</summary>

`serveRoot: '/assets'`.

</details>

### تمرين متوسط

لماذا `/api/{*path}` أوضح من `/api*` في `exclude`؟

<details>
<summary>الحل</summary>

لأن `path-to-regexp` المستخدم مع Nest يفهم البدل بصيغة `{*name}`. النمط القديم لا يطابق الشجرة التي تتوقعها.

</details>

### تمرين متقدم

`GET /assets/missing.png` مع `fallthrough: false`. من يكتب الرد؟

<details>
<summary>الحل</summary>

طبقة الملفات الثابتة تنهي الطلب. الطلب لا يصل `BooksController` ليُفسَّر كمعرف.

</details>

## أسئلة مراجعة

- من أين تُستورد `ServeStaticModule`؟
- ما فرق `rootPath` عن `serveRoot`؟
- أين تضع خيارات `maxAge`؟

## الخلاصة

`forRoot` يعلن مجلدًا وبادئة واستثناءات. الأصول تحت `/assets`. مسارات المجال تبقى للمتحكم، ومجلد الرفع خارج `rootPath`.

## ماذا بعد؟

عندما يكون `rootPath` بناء واجهة ويجلس على `/`، ماذا يحدث لمسار API غير موجود.

## المصادر الرسمية

- `@nestjs/serve-static`: https://github.com/nestjs/serve-static
- مثال Nest: https://github.com/nestjs/nest/blob/master/sample/24-serve-static/src/app.module.ts
