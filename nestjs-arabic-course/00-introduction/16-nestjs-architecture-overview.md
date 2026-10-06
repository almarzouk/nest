# Architecture الخاصة بـ NestJS

## الهدف من هذا الدرس

رسم خريطة واحدة للتطبيق ولمسار الطلب، بالأسماء التي يستخدمها التوثيق، بحيث تعرف أين سيقع كل فصل لاحق.

## المشكلة التي يحلها

من غير خريطة، كل Decorator يبدو قاعدة عشوائية. الخريطة تجيب: في أي لحظة من حياة التطبيق نكون، ومن ينادي هذه القطعة، وماذا يجب ألا تعرفه.

## الفكرة ببساطة

تطبيق Nest شجرة وحدات تبدأ من Root Module. داخل الوحدة متحكمات تستقبل الطلبات ومزوّدات تنفّذ العمل. الطلب يمر بمحطات ثابتة قبل المتحكم وبعده. المنصة تحت ذلك Express ما لم تغيّرها.

## التعريف التقني

من التوثيق:

- **Module**: صنف معلّم بـ `@Module()`. الـ Decorator يعطي Metadata تنظم بها Nest هيكل التطبيق.
- **Root Module**: الوحدة التي تمررها إلى `NestFactory.create`. منها يبدأ بناء **application graph**، وهو التركيب الداخلي الذي تحل به Nest العلاقات بين الوحدات والمزوّدات.
- **Controller**: يستقبل الطلب ويعيد الرد. منطق العمل الثقيل ليس بيته.
- **Provider**: قطعة تسجّلها الحاوية. **Service** نوع شائع من المزوّدات يضع فيه الفريق Business Logic. ليس كل Provider يسمى Service في اسم الصنف، لكن الخدمة مزوّد.
- **IoC container**: الحاوية التي تنشئ المزوّدات وتحقنها. Inversion of Control يعني أن الإطار يملك الإنشاء.
- **Request lifecycle**: التتابع الذي يعالج فيه التطبيق الطلب وينتج الرد.

سنستخدم `@Module` و `@Controller` و `@Injectable` في المعاينة التالية. تفاصيل معاملاتها فصول `04` و `05` و `06`. إن بدت غامضة، اقرأها كعناوين صناديق لا كمرجع كتابة.

## Mental Model

بناء التطبيق مرة عند الإقلاع:

```text
main.ts
   |
   v
NestFactory.create(AppModule)
   |
   v
Application graph
   |
   +-- AppModule
         imports:  UsersModule, OrdersModule, ...
         كل Module:
            controllers  -> مسارات HTTP
            providers    -> خدمات وغيرها
            exports      -> ما يسمح للغير بحقنه
   |
   v
حاوية تنشئ المزوّدات (غالبًا Singleton)
   |
   v
HTTP Adapter يربط المسارات على Express
   |
   v
listen
```

طلب واحد بعد الإقلاع، بالترتيب الذي يلخصه توثيق Request Lifecycle:

```text
1  Incoming request
2  Middleware
     2.1  Middleware عام (مثل app.use)
     2.2  Middleware مربوط بالوحدات
3  Guards
     3.1  Global
     3.2  Controller
     3.3  Route
4  Interceptors قبل المتحكم
     4.1 Global ثم 4.2 Controller ثم 4.3 Route
5  Pipes
     5.1 Global
     5.2 Controller
     5.3 Route
     5.4 Route parameter pipes
6  Controller (دالة المسار)
7  Service إن وُجدت واستُدعيت
8  Interceptors بعد المتحكم
     من Route رجوعًا إلى Controller ثم Global
9  Exception filters إن وُجد استثناء غير ممسوك
     9.1 Route ثم 9.2 Controller ثم 9.3 Global
10 Server response
```

احفظ اتجاهين فقط هذا الأسبوع:

- المحطات 2 إلى 5 من الأعم إلى الأخص غالبًا: Global ثم Controller ثم Route.
- الفلاتر عكس ذلك: تبحث من الأخص، Route أولًا. وإذا أمسك فلتر المسار الاستثناء، لا يراه الفلتر العام. الاستثناء الذي تمسكه أنت بـ `try/catch` لا يصل إلى الفلتر أصلًا.

استثناء من Middleware يذكره التوثيق: الاستثناءات الخارجة من Middleware يصل إليها طبقة الاستثناءات، لكن **الفلاتر العامة فقط** تنطبق، لأن Middleware يعمل قبل اختيار معالج المسار.

## أبسط مثال

هيكل الملفات الذي يولّده CLI، كما في First steps:

```text
src/
  main.ts
  app.module.ts
  app.controller.ts
  app.service.ts
  app.controller.spec.ts
```

| الملف | وظيفته كما يصفها التوثيق |
| --- | --- |
| `main.ts` | نقطة الدخول. تستخدم `NestFactory` لإنشاء التطبيق |
| `app.module.ts` | الوحدة الجذر |
| `app.controller.ts` | متحكم بمسار أساسي |
| `app.service.ts` | خدمة بدالة أساسية |
| `app.controller.spec.ts` | اختبار وحدة للمتحكم |

معاينة العلاقة، مبسطة عن قالب البداية. الوسوم ستُشرح لاحقًا:

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller.js';
import { AppService } from './app.service.js';

@Module({
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

```ts
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service.js';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}
```

`main.ts` كما في التوثيق الحالي:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

عند التشغيل، `GET /` يعيد النص `Hello World!`. هذا سلوك قالب البداية الموصوف في First steps.

## شرح المثال

- `@Module({ ... })` يصف الوحدة: المتحكمات التي تُنشأ، والمزوّدات التي تسجّلها الحاوية. لا يوجد `imports` لأن هذا التطبيق لا يسحب وحدة أخرى بعد.
- `controllers: [AppController]` يسجّل المتحكم. من غير هذا السطر، وجود الملف على القرص لا ينشر مسارًا.
- `providers: [AppService]` يسجّل الخدمة حتى يستطيع الحقن إيجادها.
- `@Controller()` بلا نص يعني أن مسارات هذا الصنف تبدأ من جذر التطبيق. `@Controller('books')` سيسبق المسارات بـ `/books` لاحقًا.
- `@Get()` يربط الدالة بطريقة `GET` على مسار المتحكم. المقابل في Express: `app.get('/', ...)`.
- `constructor(private readonly appService: AppService)` يعلن الاعتماد. الحاوية تمرر الخدمة. `private readonly` تصنع خاصية للصنف. الشرح اللغوي في الفصل `01`.
- `getHello()` في المتحكم لا تكتب `res.send`. تُرجع نصًا. Nest ترسل القيمة كاستجابة ناجحة: الرمز الافتراضي `200`، وطلبات `POST` تستخدم `201` ما لم تضع `@HttpCode`. هذا Standard response handling، ويقابله في Express استدعاؤك الصريح لـ `res`. فصل Controllers يفصله مع الحالة التي تحقن فيها `@Res()`.
- المتحكم لا يركّب النص بنفسه. ينادي الخدمة. حتى في مثال "أهلًا" يضع القالب العادة: HTTP في Controller، والعمل في Service.
- `@Injectable()` يعلّم الخدمة كمزوّد تُقرأ بيانات حقنه.
- `bootstrap` تبني الرسم ثم تستمع. `await bootstrap()` في أعلى الملف أسلوب ESM الذي يولّده القالب الحالي.

Express المقابل للمسار فقط:

```ts
app.get('/', (req, res) => {
  res.send('Hello World!');
});
```

Nest فرقت التسجيل (`@Module`) عن المسار (`@Get`) عن النص (الخدمة). ثلاثة ملفات لسطر Express واحد، وهذا مقصود: العادة تُبنى والبرنامج صغير، حتى لا تُكتشف الحدود أول مرة عند الملف الضخم.

## ماذا يحدث داخليًا؟

إقلاع هذا المثال:

1. Node يحمّل `main.ts` بعد الترجمة.
2. تحميل الأصناف يشغّل Decorators، فتُلصق Metadata على `AppModule` و `AppController` و `AppService`.
3. `NestFactory.create` يقرأ Metadata الوحدة الجذر.
4. يسجّل `AppService` في الحاوية وينشئها. لا اعتماديات في بنائها، فالإنشاء مباشر.
5. ينشئ `AppController` ويحقن النسخة نفسها من الخدمة.
6. المحوّل يسجّل `GET /` نحو `getHello` على Express.
7. `listen` يفتح المنفذ.

طلب `GET /`:

1. Express يستقبل الاتصال، كما في دروس Express.
2. لا Middleware إضافي في المثال، ولا Guard، ولا Pipe على معاملات. المحطات الفارغة لا تغيّر الرد.
3. تُستدعى `getHello`.
4. الدالة تنادي `appService.getHello()` على النسخة المحقونة.
5. النص يرجع إلى Nest، فتكتبه على الرد بحالة نجاح.
6. المتصفح يستقبل الجسم.

لا قاعدة بيانات، ولا صلاحية، ولا JSON. الخريطة أهم من الميزات هنا.

مصطلحات ستُعاد، بمعنى ضيق يكفي للخريطة:

| المصطلح | جملة واحدة |
| --- | --- |
| Application context | التطبيق وهو يعمل، حتى من غير HTTP في بعض أشكال Standalone. هنا السياق يشمل خادم HTTP |
| Dependency graph | من يعتمد على من داخل الحاوية. في المثال: المتحكم يعتمد على الخدمة، والخدمة لا تعتمد على أحد |
| Singleton | النسخة الواحدة من `AppService` تُخدم كل الطلبات. هذا الافتراضي المسمى DEFAULT scope |
| HTTP Adapter | الجسر الذي سجل `GET /` على Express |

## مثال عملي

طلب `POST /orders` على تطبيق نما بعد هذا القالب، كتمرين تتبع لا كتنفيذ:

```text
Client
  POST /orders   JSON
      |
      v
Express يستقبل
      |
      v
Middleware: تحليل JSON إن كان مسجّلًا عند الحافة
      |
      v
Guard: هل الطلب يحمل هوية مقبولة؟
      |  إن رفض: يرد 401/403 والمسار يتوقف
      v
Interceptor قبل: يبدأ قياس الزمن
      |
      v
Pipe: يتحقق من DTO الكمية والكتاب
      |  إن فشل: 400 والمسار يتوقف
      v
OrdersController.create
      |
      v
OrdersService تنفّذ القاعدة وتنادي طبقة البيانات
      |
      v
Interceptor بعد: يضيف زمن الاستجابة
      |
      v
201 و JSON
```

إذا رمت الخدمة خطأ لم يُمسك، يقفز التنفيذ إلى Exception Filter الأخص المتاح، ويرجع شكل خطأ متفق عليه بدل إسقاط الاتصال.

لاحظ ما لا تعرفه الخدمة: لا طريقة HTTP، ولا Express. تعرف كتابًا وكمية ومستخدمًا مُرّروا إليها. هذا هو loosely coupled الذي وعد به التوثيق، مترجمًا إلى حدود دالة.

## أخطاء شائعة

- حفظ ترتيب المحطات من مقالات قديمة. المصدر الذي نعتمده هو صفحة Request Lifecycle الحالية، والترتيب فيها: Middleware، ثم Guards، ثم Interceptors، ثم Pipes، ثم المتحكم، ثم Interceptors في طريق العودة، والفلاتر عند الاستثناء.
- وضع SQL أو حساب السعر داخل `getHello` لأن المثال صغير. المثال صغير ليُظهر الحدود، لا ليبرر إلغاءها.
- نسيان تسجيل المتحكم أو المزوّد في `@Module` والاكتفاء بوجود الملف.
- إرجاع `res.send` من Express داخل كل دالة لأنك ترتاح له. ذلك Library-specific style ستتعلم متى تحتاجه. العادة الافتراضية: أرجع قيمة.
- اعتبار Service والـ Provider كلمتين مترادفتين دائمًا. الخدمة مزوّد. المزوّد قد يكون قيمة أو مصنعًا أو صنفًا آخر. فصل Providers يفتح هذا.

## متى نستخدم هذه الخريطة؟

في كل مرة تسأل "أين أضع هذا الكود؟" قبل فتح محرر. إذا كان الجواب غير واضح، الكود غالبًا يخلط طبقتين.

## متى لا نضخ كل المحطات؟

مثال `Hello World` لا يحتاج Guard ولا Pipe. المحطة تُضاف عندما توجد مسؤولية. إطار ذو رأي يحدد المكان، وأنت تحدد إن كانت المسؤولية موجودة. إضافة كل الزينة من أول ملف هي Overengineering مبكر.

## مقارنة مع Express

```text
Express                         Nest
app.get('/users', handler)      @Controller('users') + @Get()
middleware قبل المعالج          Middleware في أول مسار الطلب
if داخل المعالج للصلاحية        Guard في محطته
if داخل المعالج للشكل           Pipe في محطته
منطق و SQL داخل المعالج         Service وطبقة بيانات
new داخل المعالج                تسجيل في providers وحقن
شكل خطأ مختلف لكل مسار          Exception Filter
res.send في المعالج             إرجاع قيمة، والإطار يكتب الرد
```

## تمارين

### تمرين سهل

رتب هذه الأسماء كما في مسار الطلب الناجح: Pipe، Controller، Middleware، Guard، Interceptor قبل المتحكم.

<details>
<summary>الحل</summary>

Middleware، ثم Guard، ثم Interceptor قبل المتحكم، ثم Pipe، ثم Controller.

</details>

### تمرين متوسط

في معاينة الكود، ماذا يحدث لو حذفت `AppService` من `providers` وأبقيت المعامل في constructor؟ صف النتيجة المتوقعة بلغة الحاوية، لا بنص الاستثناء الحرفي.

<details>
<summary>الحل</summary>

الحاوية لا تجد مزوّدًا مسجّلًا يطابق اعتماد المتحكم، فتفشل عملية إنشاء التطبيق قبل أن يخدم الطلب. وجود الملف والاستيراد في TypeScript لا يغنيان عن التسجيل في الوحدة.

</details>

### تمرين متقدم

استثناء رُمي من Middleware عام، واستثناء رُمي من Service بعد أن اختير المسار. أي فلاتر يحق لها رؤية كل منهما حسب التوثيق؟

<details>
<summary>الحل</summary>

استثناء Middleware تراه الفلاتر العامة فقط، لأن معالج المسار لم يُختر بعد. استثناء الخدمة، إن بقي غير ممسوك، يدخل ترتيب الفلاتر من الأخص: Route ثم Controller ثم Global، ويمسكه أول فلتر منطبق ولا يُمرَّر إلى الذي بعده.

</details>

## أسئلة مراجعة

- من أين يبدأ application graph؟
- ما الفرق بين تسجيل Controller في الملف وتسجيله داخل `@Module`؟
- لماذا تُرجع `getHello` نصًا بدل أن تنادي `res.send`؟
- ما اتجاه اختيار Exception Filter مقارنة بـ Guard؟
- أين تعيش قاعدة العمل في الخريطة؟

## الخلاصة

الإقلاع يبني رسم وحدات وحاوية ثم يربط HTTP. الطلب يمر بـ Middleware ثم Guard ثم Interceptor ثم Pipe ثم Controller ثم Service، ويعود عبر Interceptors، وتدخل الفلاتر عند الاستثناء من الأخص إلى الأعم. Controller يستقبل، و Service تعمل، و Module يرسم الحدود.

## ماذا بعد؟

الخريطة تكفي للأسماء. في [الأمثلة](./examples.md) نطبقها على متجر صغير من غير بناء مشروع، ثم [تمارين الفصل](./exercises.md)، ثم [التمهيد لـ TypeScript](./next-step.md) حيث نفك `constructor(private readonly ...)` قبل أن ننشئ مشروعًا.

## المصادر الرسمية

- NestJS — Introduction: https://docs.nestjs.com/
- NestJS — First steps: https://docs.nestjs.com/first-steps
- NestJS — Modules: https://docs.nestjs.com/modules
- NestJS — Controllers: https://docs.nestjs.com/controllers
- NestJS — Providers: https://docs.nestjs.com/providers
- NestJS — Request lifecycle: https://docs.nestjs.com/faq/request-lifecycle
- NestJS — Platform agnosticism: https://docs.nestjs.com/fundamentals/platform-agnosticism
