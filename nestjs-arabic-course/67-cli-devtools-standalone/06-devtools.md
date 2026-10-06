# Nest Devtools

## الهدف من هذا الدرس

ربط رف المحلي بـ Nest Devtools لرؤية الوحدات والمسارات وخطأ الحقن، وإبقاء خادم الاستكشاف خارج الإنتاج.

## المشكلة التي يحلها

«Cannot resolve dependency» على سلسلة طويلة بلا أن يوضح أي وحدة نقصها الاستيراد. أو مسار `POST /projects` لا يُظهر ترتيب الحارس والأنبوب. أو `DevtoolsModule` يعمل في الإنتاج فيفتح منفذ استكشاف.

## الفكرة ببساطة

`snapshot: true` في `NestFactory.create` يجمع وصف الرسم مرة عند الإقلاع. `DevtoolsModule.register({ http: process.env.NODE_ENV !== 'production' })` يفتح خادمًا خفيفًا على المنفذ 8000 للوحة فقط في التطوير. الرسم يعرض الوحدات والصفوف والمسارات. فشل الإقلاع مع `abortOnError: false` يكتب `graph.json` عبر `PartialGraphHost` لتُسحب اللوحة في وضع المعاينة.

## التعريف التقني

`npm i @nestjs/devtools-integration`. في `main.ts`:

```ts
const app = await NestFactory.create(AppModule, {
  snapshot: true,
});
```

`http: false` لا يعرض شيئًا حتى لو بقي الاستيراد. المنفذ يُغيَّر بخيار `port`. كل وحدة تتصل بـ `InternalCoreModule` العام؛ إخفاء الوحدات العامة في الشريط ينظّف الصورة. Routes explorer يشمل HTTP و WebSocket و gRPC و GraphQL. Playground يطلب رمز sandbox تطبعه العملية عند الإقلاع. `SerializedGraph` من `@nestjs/core` يكتب لقطة بعد `listen` أو `init` بـ `writeFileSync`. الصفحة تطلب ألا يعمل `DevtoolsModule` في الإنتاج.

## Mental Model

```text
إقلاع محلي + snapshot → رسم
منفذ 8000 في التطوير فقط
فشل حقن → graph.json جزئي
```

## أبسط مثال

```ts
import { Module } from '@nestjs/common';
import { DevtoolsModule } from '@nestjs/devtools-integration';

@Module({
  imports: [
    DevtoolsModule.register({
      http: process.env.NODE_ENV !== 'production',
    }),
  ],
})
export class AppModule {}
```

ثم `npm run start:dev` وافتحوا لوحة Devtools. ابحثوا عن `ProjectsController`.

## شرح المثال

- الشرط على `NODE_ENV` يطفئ HTTP خارج التطوير.
- جمع اللقطة مرة عند الإقلاع. معالجة الطلب العادية لا تعيد بناء الرسم لكل نداء.
- Hide global modules يخفي حافة `InternalCoreModule` نحو كل الوحدات.

## ماذا يحدث داخليًا؟

1. `snapshot` يخزن وصف المزودات والمتحكمات أثناء البناء.
2. `DevtoolsModule` يخدم هذا الوصف على منفذه.
3. Routes explorer يحل ترتيب الحراس والاعتراضات والأنابيب كما رُبطت، لا كما تتوقعون من قراءة الزخارف.
4. عند فشل الحقن، `abortOnError: false` يكمل بما يكفي لكتابة الرسم الجزئي:

```ts
import { writeFileSync } from 'node:fs';
import { PartialGraphHost } from '@nestjs/core';

bootstrap().catch((err) => {
  writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
  process.exit(1);
});
```

5. سحب `graph.json` في وضع Preview يبرز الوحدة التي ينقصها استيراد، مثل `DiagnosticsModule` في مثال الصفحة.

## مثال عملي

`ProjectsService` يطلب `PlanPolicy` والوحدة لا تستورد `BillingModule`. الرسالة النصية تذكر الرمز الناقص. الرسم يظهر السلسلة حتى `ProjectsModule`. الإصلاح استيراد الوحدة التي تصدّر السياسة، لا `forwardRef` عشوائي. في Audit راقبوا مزودًا بنطاق طلب يمر عبر شجرة واسعة؛ ذلك يعود إلى كلفة النطاق في فصل الأداء. Playground على مسار إنشاء المشروع يستخدم رمز الجلسة المطبوع ولا يُترك مفتوحًا على قاعدة مشتركة.

## أخطاء شائعة

- `http: true` دائمًا فيصل الاستكشاف إلى نشر عام.
- نسيان `snapshot: true` فتفرغ اللوحة.
- `abortOnError` يبقى `true` فلا يُكتب `graph.json`.
- اعتبار اللوحة بديلًا عن اختبار عزل المستأجر.

## متى نستخدمه؟

محليًا لفهم الرسم، وتشخيص الحقن، ومراجعة مسار معقّد، وقياس زمن إنشاء الصفوف عند إقلاع بطيء.

## متى لا نستخدمه؟

في صورة الإنتاج. ولا لتجاوز المصادقة على API الحقيقي؛ رمز Playground جلسة لوحة تطوير.

## مقارنة

| العرض | ماذا يجيبه |
| --- | --- |
| الرسم | من يستورد من |
| Routes | ترتيب الحارس والأنبوب على المسار |
| Audit | شكل معماري يستحق النظر |
| `graph.json` | إقلاع فشل قبل اكتمال الرسم |

## تمارين

### تمرين سهل

أي خياران يمنعان ظهور الاستكشاف في الإنتاج؟

<details>
<summary>الحل</summary>

`http` مربوط بشرط ليس إنتاجًا، وعدم تشغيل الوحدة هناك أصلًا. `snapshot` وحده يجمع وصفًا؛ الخادم هو `http`.

</details>

### تمرين متوسط

لماذا `PartialGraphHost` مع `abortOnError: false`؟

<details>
<summary>الحل</summary>

الإقلاع يموت قبل رسم كامل. `abortOnError: false` يتيح كتابة الجزء الذي بُني، واللوحة تظهر أين انقطع الحقن.

</details>

### تمرين متقدم

مسار `POST /projects` يعمل بلا حارس المنظمة في اللوحة بينما الزخرفة على المتحكم. ماذا تقرأون؟

<details>
<summary>الحل</summary>

Routes explorer يرتب ما رُبط فعلًا. قد يكون الحارس على وحدة غير مستوردة أو مغلوبًا بترتيب التسجيل. اللوحة تعرض السلسلة المحلولة لا نية الزخرفة فقط.

</details>

## أسئلة مراجعة

- أي حزمة وأيهما من `@nestjs/core`؟
- ما المنفذ الافتراضي؟
- متى يُجمع الرسم؟
- أين لا يعمل `http`؟

## الخلاصة

Devtools يعرض رسم رف ومساراته حين يكون `snapshot` مفعلًا والخادم محصورًا بالتطوير. فشل الحقن يُشخَّص بـ `graph.json`. الإنتاج بلا هذا المنفذ.

## ماذا بعد؟

مهمة تصدير بلا HTTP تستخدم سياق تطبيق مستقل. الدرس التالي Standalone.

## المصادر الرسمية

- NestJS — Devtools: https://docs.nestjs.com/devtools/overview
