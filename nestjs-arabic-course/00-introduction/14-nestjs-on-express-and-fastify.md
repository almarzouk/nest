# NestJS فوق Express و Fastify

## الهدف من هذا الدرس

فهم أن Nest طبقة معمارية، وأن HTTP الفعلي يمر عبر محوّل منصة: Express افتراضيًا، و Fastify عندما تختاره.

## المشكلة التي يحلها

لو كُتب كل Controller بنداءات خاصة بـ Express فقط، صار الانتقال إلى خادم أسرع، أو إضافة WebSockets، إعادة كتابة للمنطق. نريد كود قواعد ومسارات يتحدث إلى Nest، ونريد بابًا نصل منه إلى Express عندما نحتاج مكتبة لا تعرف Nest.

## الفكرة ببساطة

Nest لا تستمع إلى المنفذ ببروتوكول اخترعته. هي تبني تطبيقك ثم تسلّم HTTP إلى منصة. المنصة الافتراضية Express عبر `@nestjs/platform-express`. المنصة البديلة المدمجة Fastify عبر `@nestjs/platform-fastify`.

بين كودك والمنصة يوجد **HTTP Adapter**: قطعة تترجم "سجّل هذا المسار" و "مرر هذا الـ middleware" إلى نداءات المكتبة التي تحتها.

## التعريف التقني

من فصل Performance في التوثيق:

افتراضيًا تستخدم Nest إطار Express. هي متوافقة مع مكتبات أخرى مثل Fastify. الاستقلال يتم عبر framework adapter عمله الأساسي تمرير middleware والمعالجات إلى التنفيذ الخاص بكل مكتبة.

Fastify بديل مناسب لأنه يحل مسائل التصميم بطريقة قريبة من Express، وهو أسرع بشكل ملحوظ. التوثيق يقول إنه يحقق قرابة ضعف نتائج القياس. Nest تُبقي Express افتراضيًا لأن Express واسع الانتشار، ومعروف، ومجموعة الـ middleware المتوافقة معه ضخمة ومتاحة لمستخدمي Nest مباشرة.

من First steps: Nest مصممة لتكون platform-agnostic. أجزاء منطقية قابلة لإعادة الاستخدام عبر أنواع تطبيقات مختلفة. أي إطار HTTP لـ Node يمكن أن يعمل إذا كُتب له محوّل. المدعومان جاهزين: Express و Fastify.

كل منصة لها واجهة: `NestExpressApplication` و `NestFastifyApplication`. تمرير النوع إلى `NestFactory.create` اختياري، وتفعله عندما تريد دوال تلك المنصة على كائن `app`.

## Mental Model

```text
Controller / Guard / Pipe / Service
              |
              |  كودك يتحدث إلى عقود Nest
              v
         Nest core
              |
              |  HTTP Adapter يترجم
              v
     Express   أو   Fastify
              |
              v
           node:http وعائلة الخادم
              |
              v
            المنفذ
```

من الأعلى إلى الأسفل: قرارك المعماري، ثم ترجمة الإطار، ثم مكتبة HTTP، ثم Node.

## أبسط مثال

الافتراضي لا يذكر Express في `main.ts`:

```ts
const app = await NestFactory.create(AppModule);
```

هنا النوع العام `INestApplication`. تحته Express لأن الحزمة الافتراضية هي `@nestjs/platform-express`. لا تستورد Express حتى تحتاجه.

عندما تريد دوال Express على `app`، التوثيق يكتب:

```ts
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

القوس الزاوي نوع TypeScript فقط: يغيّر ما يقترحه المحرر على `app`. لا يبدّل المنصة وحده. تبديل المنصة إلى Fastify يتم بتمرير محوّل، في فصل Fastify، بالشكل الذي يوثّقه Nest:

```ts
const app = await NestFactory.create(AppModule, new FastifyAdapter());
```

**لا تركّب Fastify في هذا الدرس.** السطر هنا ليشرح فكرة المحوّل: المعامل الثاني يختار المنصة. `FastifyAdapter` صنف من `@nestjs/platform-fastify`. التفاصيل، بما فيها الاستماع على `0.0.0.0` لأن Fastify يقيّد الاستماع افتراضيًا، تعيش في فصل الأداء عندما يصبح القياس سؤالًا حقيقيًا.

## شرح المثال

- `NestFactory.create(AppModule)` يبني التطبيق والمنصة الافتراضية معًا.
- غياب اسم Express من الملف مقصود: معظم الدروس لا تحتاج كائن Express.
- `NestExpressApplication` بوابة هروب معلنة. التوثيق يقول: لست مضطرًا إلى تحديد النوع إلا إذا أردت API المنصة.
- `new FastifyAdapter()` يخبر المصنع أي محوّل يستخدم. بقية التطبيق — Modules و Controllers — تبقى. هذا معنى "طبقة فوق الإطار".

حرية التوثيق المهمة: Nest تعرض APIs المنصة مباشرة، حتى تستخدم مكتبات Express أو Fastify الموجودة في المجتمع. الطبقة الإضافية ليست سجنًا.

## ماذا يحدث داخليًا؟

عند تسجيل مسار في Controller:

1. Nest تحفظ Metadata: الطريقة، والمسار، والدالة.
2. عند الإقلاع، المحوّل يمر على هذه البيانات ويسجّل على Express ما يعادل `app.get` أو `app.post`، أو على Fastify ما يعادله في تلك المكتبة.
3. عند وصول طلب، المنصة تدخل مكدسها أولًا.
4. Middleware المسجّل عالميًا عبر `app.use()` يعمل في هذه الطبقة المبكرة. توثيق Request Lifecycle يبدأ المسار من هنا ثم Middleware الخاص بالوحدات.
5. بعدها Guards ثم Interceptors ثم Pipes ثم الدالة. هذا ترتيب Nest فوق المنصة، لا ترتيب Express وحده.
6. الرد يرجع عبر المحوّل إلى كائن الرد الخاص بالمنصة.

لذلك Middleware في Nest قريب جدًا من Middleware في Express على المنصة الافتراضية. Guard ليس Middleware باسم ألطف؛ هو محطة لاحقة لها عقد مختلف (`CanActivate`). ستُبنى هذه الجملة مرة أخرى عندما نكتب الكود.

فروق عملية يذكرها توثيق Fastify حتى لا تفاجأ يوم التبديل:

- وصفات مكتوبة لـ Express قد لا تعمل. تستخدم مقابلًا لـ Fastify.
- التحويل (redirect) يُكتب بأسلوب Fastify: `status` ثم `redirect`.
- Middleware هناك تستقبل `req` و `res` الخام، لأن Nest تستخدم `@fastify/middie` تحت الغطاء.
- Fastify يسمع على `localhost` فقط إلى أن تمرر `0.0.0.0` في `listen` إن أردت واجهات أخرى.

هذه فروق منصة، لا فروق في قواعد عملك داخل Service.

فصل Platform agnosticism يوسع الصورة: الأجزاء المنطقية نفسها يمكن أن تخدم HTTP، أو Microservices، أو WebSockets، ويوجد GraphQL كطبقة API أخرى. "ابنِ مرة واستخدم في أكثر من مكان" هدف، ويتحقق للقطع التي لا تستورد `Request` الخاص بـ Express داخل قاعدة العمل. لهذا سنبقي `req` على حافة Controller.

## مثال عملي

مكتبة تقيس الطلبات وتعرف Express فقط. على المنصة الافتراضية تستطيع وصلها من مكان الإقلاع عبر واجهة Express، لأن Nest تعطيك الوصول إلى المنصة. الشكل المفهوم الآن:

```text
إقلاع Nest
   -> الوصول إلى تطبيق Express التحتي
   -> تسجيل middleware تلك المكتبة
   -> طلبات HTTP تمر عليها قبل مساراتك
```

لن نكتب نداء `getHttpAdapter()` هنا حتى لا يصبح تعويذة. اسم الفكرة: الهروب إلى المنصة قرار واعٍ على الحافة، لا استيراد `express` داخل كل Service.

إذا كان القياس لاحقًا يقول إن وقت الإطار HTTP هو عنق الزجاجة، Fastify مرشح يوثّقه Nest. إذا كانت مكتبات الفريق كلها middleware لـ Express، البقاء على الافتراضي هو القرار المتسق مع سبب اختيار Nest لـ Express كافتراضي.

## أخطاء شائعة

- استيراد `express` وإنشاء `express()` بجانب `NestFactory.create` للحصول على "الاثنين". ينتج خادمان وصيانة مزدوجة. الوصول يكون عبر تطبيق Nest لا عبر تطبيق ثانٍ.
- كتابة `req.cookies` و `req.session` داخل Service. الخدمة صارت تعرف Express، واختبارها صار يحتاج طلبًا مزيفًا. المنصة تبقى عند الحافة.
- تبديل Fastify في أول أسبوع بحثًا عن "الضعف في القياس" قبل وجود قياس. التوثيق نفسه يبرر البقاء على Express بالمنظومة لا بالسرعة.
- افتراض أن كل `app.use` من مدونة Express يعمل حرفيًا على Fastify. المحوّل يترجم ما تستطيع Nest ترجمته، والوصفات الخاصة بالمنصة تُراجع.

## متى نبقى على Express؟

الافتراضي في هذه الدورة كلها إلى أن يظهر سبب. السبب النموذجي للبقاء: مكتبات middleware، وأمثلة المجتمع، وغياب قياس يقول إن الخادم HTTP هو العائق.

## متى نختار Fastify؟

عندما تكون سرعة طبقة HTTP مطلبًا مثبتًا، وتقبل مراجعة الوصفات المرتبطة بـ Express. الفصل `53` يعود إلى ذلك مع بقية الأداء. الاختيار يتم في `NestFactory.create`، لا بإعادة كتابة الـ Controllers.

## مقارنة

| | Express عبر Nest | Fastify عبر Nest | استدعاء Express وحدك |
| --- | --- | --- | --- |
| الحزمة | `@nestjs/platform-express` وهي الافتراضية | `@nestjs/platform-fastify` | `express` |
| من يسجّل مسارات Controllers | المحوّل | المحوّل | أنت بـ `app.get` |
| Middleware المجتمعية | الأوسع | تحتاج مقابلًا غالبًا | واسعة، بلا DI |
| سرعة الطبقة كما يصفها توثيق Nest | الأساس | أعلى في قياساتهم، قرابة الضعف | لا علاقة لـ Nest |
| كود Service | لا يعرف المنصة | لا يعرف المنصة | غالبًا يعرف `req` إن أهملت الفصل |

Express المباشر:

```ts
app.get('/users', handler)
```

Nest على أي من المنصتين:

```ts
@Get()
findAll() {}
```

الفرق ليس اسم الدالة. الفرق أن Nest تملك التسجيل، والمنصة تنفذه.

## تمارين

### تمرين سهل

ما المنصة التي يعمل عليها `NestFactory.create(AppModule)` إذا لم تمرر محوّلًا، وما اسم الحزمة؟

<details>
<summary>الحل</summary>

Express، عبر `@nestjs/platform-express`.

</details>

### تمرين متوسط

لماذا تمرير `<NestExpressApplication>` لا يكفي وحده ليحوّل مشروعًا إلى Fastify؟

<details>
<summary>الحل</summary>

القوس الزاوي معلومة نوع للمحرر والمترجم. المحوّل الفعلي هو ما يمر في `create`، مثل `new FastifyAdapter()`. من غير محوّل Fastify تبقى المنصة Express، والنوع لو كذب سيصف دوالًا غير الموجودة على التطبيق الحقيقي.

</details>

### تمرين متقدم

Service تستورد `Request` من Express لتقرأ ترويسة. ماذا تخسر بالنسبة لوعد platform-agnostic، وما المكان الأقرب لقراءة الترويسة؟

<details>
<summary>الحل</summary>

تخسر تشغيل القاعدة نفسها على Fastify أو في سياق غير HTTP من غير تعديل الخدمة، ويصبح الاختبار محتاجًا كائن طلب Express. المكان الأقرب هو حافة Controller أو Decorator مخصص للمعامل يمرر القيمة الجاهزة إلى الخدمة. الخدمة تستقبل نصًا أو كائنًا بسيطًا. بناء الـ Decorator درس في فصل Custom Decorators، والفكرة المعمارية تكفي الآن.

</details>

## أسئلة مراجعة

- ما وظيفة HTTP Adapter؟
- لماذا Express هو الافتراضي إذا كان التوثيق يصف Fastify بأنه أسرع؟
- متى تستورد نوع `NestExpressApplication`؟
- أي جزء من تطبيقك يجب ألا يعرف اسم المنصة؟

## الخلاصة

Nest طبقة فوق خادم HTTP. Express الافتراضي، و Fastify محوّل بديل. كود الوحدات والخدمات يبقى، والهروب إلى API المنصة يتم عند الحافة عندما تحتاج مكتبة مجتمع تلك المنصة.

## ماذا بعد؟

التوثيق يقول إن المعمارية مستوحاة من Angular. الدرس التالي يحدد ما الذي يشبه Angular فعلًا، حتى لا تظن أن الدورة ستمر على المتصفح.

## المصادر الرسمية

- NestJS — First steps (Platform): https://docs.nestjs.com/first-steps
- NestJS — Performance (Fastify): https://docs.nestjs.com/http/performance
- NestJS — Platform agnosticism: https://docs.nestjs.com/fundamentals/platform-agnosticism
- NestJS — HTTP adapter: https://docs.nestjs.com/faq/http-adapter
- Express — Hello world: https://expressjs.com/en/starter/hello-world.html
