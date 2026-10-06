# دالة bootstrap

## الهدف من هذا الدرس

قراءة دالة الإقلاع في `main.ts` كسلسلة: ابنِ التطبيق ثم ابدأ الاستماع.

## المشكلة التي يحلها

لو وُضع `listen` قبل أن تُسجَّل المسارات والمزوّدات، يصل طلب إلى منفذ مفتوح وتطبيق غير جاهز. الإقلاع دالة واحدة تُكمل البناء أولًا.

## الفكرة ببساطة

`bootstrap` دالة غير متزامنة تنشئ التطبيق من الوحدة الجذر ثم تفتح المنفذ. في قالب ESM تُستدعى بـ `await bootstrap()` في أعلى الملف.

## التعريف التقني

من First steps: الملف يستورد `NestFactory` من `@nestjs/core` و `AppModule`، ثم:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
await bootstrap();
```

`create` يعيد كائنًا يطبّق `INestApplication`. دوال هذا الكائن تُشرح في الفصول التالية. هنا هو نتيجة البناء.

## Mental Model

```text
تحميل main.ts
    -> await create(AppModule)   بناء الرسم والحاوية والمسارات
    -> await listen(port)        فتح المنفذ
    -> العملية تبقى حية
```

## أبسط مثال

الكود أعلاه هو المثال الرسمي الحالي. لا تختصره بحذف `await`.

## شرح المثال

- الاستيرادان يحمّلان المصنع والوحدة الجذر. مسار `.js` أسلوب ESM كما في فصل التثبيت.
- `async` تسمح بـ `await` داخل الدالة.
- `create` ترجع Promise لأن البناء قد ينتظر تهيئة مزوّدات غير متزامنة لاحقًا.
- `??` يختار `3000` عندما `PORT` قيمة `null` أو `undefined`. نص فارغ لا يُستبدل بهذا المعامل؛ لا تضع `PORT=` فارغًا وتتوقع 3000.
- `await bootstrap()` يشغّل الإقلاع ويمنع أن تنتهي الوحدة قبل اكتمال الـ Promise في أسلوب ESM.

## ماذا يحدث داخليًا؟

تحميل الملف يشغّل Decorators على الأصناف المستوردة عبر `AppModule`. ثم `create` تمشي الرسم وتبني الحاوية وتربط HTTP. `listen` بعد ذلك يستدعي استماع المنصة. إن رُفضت أي Promise ولم تُمسك، الإقلاع يفشل. الافتراضي عند خطأ إنشاء التطبيق إنهاء العملية برمز `1`.

## مثال عملي

أضف سطرًا بعد `listen` فقط عندما تحتاج إعدادًا على `app` قبل الاستماع، مثل أنبوب عام:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
```

الأنبوب العام مكانه **قبل** `listen` حتى لا يفتح المنفذ قبل تركيب الحماية. ستضيف `useGlobalPipes` في فصل Validation. اليوم لا تضفه قبل أن تعرف DTO.

## أخطاء شائعة

- استدعاء `bootstrap()` من غير `await` في ESM ثم الاعتقاد أن الأخطاء ظاهرة دائمًا.
- وضع منطق عمل داخل `bootstrap`. المدخل يركّب الإطار فقط.
- قراءة `PORT` بعد `listen` لتغيير المنفذ. الاختيار يحدث في الوسيط.

## متى نستخدم هذه الدالة؟

في كل تطبيق HTTP في الدورة. الاسم `bootstrap` عرف التوثيق لا كلمة سحرية.

## متى لا نستخدم listen؟

في تطبيق Standalone بلا HTTP، مثل أمر CLI لاحقًا. هناك سياق تطبيق من غير استماع. الفصل `67` يذكره. الآن نبقى على HTTP.

## مقارنة

```text
Express:  const app = express(); app.listen(3000)
Nest:     const app = await NestFactory.create(AppModule); await app.listen(...)
```

الفرق: Nest تبني وحدات وحاوية قبل الاستماع. Express ينتظر أن تسجّل أنت المسارات على `app` قبل `listen` أو بعده حسب أسلوبك، والقالب المرتب يسجّل قبل الاستماع أيضًا.

## تمارين

### تمرين سهل

ماذا يحدث أولًا: `create` أم `listen`؟

<details><summary>الحل</summary>

`create` ثم `listen`.

</details>

### تمرين متوسط

لماذا الدالة `async`؟

<details><summary>الحل</summary>

لأن `create` و `listen` ترجعان Promise ويجب انتظار اكتمال البناء والاستماع.

</details>

### تمرين متقدم

`PORT` غير موجود في البيئة. أي منفذ يُستخدم، وأي قيمة لا يعوّضها `??`؟

<details><summary>الحل</summary>

يُستخدم `3000` لأن المتغير `undefined`. سلسلة فارغة ليست `null` ولا `undefined`، فلا يستبدلها `??`.

</details>

## أسئلة مراجعة

- من أين يُستورد `NestFactory`؟
- ماذا يُمرَّر إلى `create`؟
- لماذا ننتظر `create` قبل `listen`؟

## الخلاصة

`bootstrap` تبني التطبيق من `AppModule` ثم تفتح المنفذ. لا عمل مجال داخل هذه الدالة.

## ماذا بعد؟

`NestFactory` نفسه: ماذا يقدر أن ينشئ.

## المصادر الرسمية

- NestJS — First steps: https://docs.nestjs.com/first-steps
