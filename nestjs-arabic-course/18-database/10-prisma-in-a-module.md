# وضع Prisma داخل Module

## الهدف من هذا الدرس

تسجيل `PrismaService` في `PrismaModule` وتصديرها حتى تستوردها ميزات الكتب والإعارات من غير إنشاء عميل ثانٍ.

## المشكلة التي يحلها

وضع `PrismaService` في `providers` داخل كل ميزة ينشئ نسخة لكل وحدة إن لم تُشارَك، أو يفشل الحقن إن نُسيت من `exports`. المتحكم الذي يستورد العميل المولَّد مباشرة يكسر حد الخدمة.

## الفكرة ببساطة

وحدة صغيرة تملك المزوّد وتصدّره. `BooksModule` تستورد `PrismaModule` وتسجّل `BooksService` فقط. النسخة الواحدة، لأن المزوّد singleton افتراضيًا، تعني مجمع اتصالات واحدًا.

## التعريف التقني

من صفحة Nest:

```ts
import { Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service.js";

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

```ts
@Module({
  imports: [PrismaModule],
  controllers: [BooksController],
  providers: [BooksService],
})
export class BooksModule {}
```

`ConfigModule` عام من فصل الإعداد حتى يحقن `PrismaService` خدمة `ConfigService` في بنّائها. إن لم يكن عامًا، `PrismaModule` تستورد `ConfigModule`.

`@Global()` على `PrismaModule` مع استيرادها مرة في `AppModule` يجعل الخدمة مرئية في كل مكان. الصفحة الرسمية تشير إلى فصل Global modules للمقايضة: الراحة مقابل إخفاء من يعتمد على القاعدة. في مكتبة صغيرة الاستيراد الصريح أوضح. إن كثرت الوحدات التي تكتب جداول، العام قرار واعٍ لا عادة.

اختبار وحدة يستبدل الرمز `PrismaService` بـ `useValue`. اختبار وحدة تستورد `PrismaModule` الحقيقية يستخدم `overrideProvider(PrismaService)` حتى لا يعمل `$connect`.

لا تصدّر `PrismaClient` كرمز ثانٍ. الخدمة الممتدة هي الرمز الوحيد.

## Mental Model

```text
AppModule
  ConfigModule (عام)
  BooksModule
    imports PrismaModule
      exports PrismaService
    BooksService تحقن PrismaService
```

## أبسط مثال

```ts
import { Global, Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service.js";

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

```ts
@Module({
  imports: [PrismaModule, BooksModule, LoansModule],
})
export class AppModule {}
```

## شرح المثال

`@Global()` هنا اختياري ومُعلَّم. بعد تسجيل `PrismaModule` في الجذر، `BooksService` و `LoansService` تحقن الخدمة ولو لم تذكر وحداتهما `PrismaModule`. إن حذفت `exports` بقي الاختباء رغم `@Global()`.

ابدأ بالاستيراد الصريح في المثال الأول إن كان عدد الوحدات قليلًا. انتقل إلى `@Global()` حين يتكرر الاستيراد في كل ملف وحدة بلا استثناء.

## ماذا يحدث داخليًا؟

Nest ينشئ `PrismaService` مرة للمستوردين الذين يرون المزوّد نفسه، ويستدعي `onModuleInit` عند إقلاع التطبيق لا عند كل طلب. `BooksService` تستقبل تلك النسخة في البنّاء. دورة الطلب لا تفتح اتصالًا جديدًا. المجمع داخل المحوّل يعير اتصالًا للاستعلام ويعيده.

إن سجّلت الخدمة مرتين كمزوّد محلي في وحدتين بلا تصدير مشترك، تحصل على عميلين ومجمعين. شارك المزوّد بالتصدير أو بالوحدة العامة.

## مثال عملي

`LoansModule` تستورد `PrismaModule` وتوفّر `LoansService` التي تستدعي `$transaction`. `BooksModule` لا تستورد `LoansModule` كي لا تصير الإعارة تفصيلًا داخليًا للكتالوج. كلتاهما تعتمد على القاعدة عبر الوحدة المشتركة، لا على بعضهما بسبب `PrismaService`.

اختبار `BooksService` لا يستورد الوحدة العامة الحقيقية. يمرّر مزيفًا. اختبار e2e يستورد `AppModule` ويستدعي `app.close()`.

## أخطاء شائعة

- مزوّد `PrismaService` مكرر في كل `providers`.
- `@Global()` من غير استيراد الوحدة في الجذر.
- حقن `PrismaService` في المتحكم وكتابة الاستعلام هناك.
- نسيان أن `ConfigModule` يجب أن يكون مرئيًا لبنّاء الخدمة.

## متى نستخدمه؟

حالما توجد أكثر من خدمة تقرأ PostgreSQL عبر Prisma.

## متى لا نستخدمه؟

في سكربت CLI خارج Nest يتصل بأمر واحد ثم يخرج. هناك عميل محلي يُغلَق في `finally` أوضح من وحدة.

## مقارنة

| استيراد صريح | `@Global()` |
| --- | --- |
| كل ميزة تذكر `PrismaModule` | استيراد الجذر مرة |
| الاعتماد ظاهر | الاعتماد مخفي |
| مناسب لعدد قليل | مناسب حين القاعدة في كل الميزات |

## تمارين

### تمرين سهل

ماذا يجب أن تحتوي `exports` حتى ترى `BooksService` العميل؟

<details>
<summary>الحل</summary>

`PrismaService`. وجودها في `providers` وحدها يبقيها داخل `PrismaModule`.

</details>

### تمرين متوسط

لماذا لا تُوضع `PrismaService` في `providers` الخاصة بـ `BooksModule` و `LoansModule` معًا؟

<details>
<summary>الحل</summary>

كل وحدة تنشئ نسخة، فيتعدد المجمع. التصدير من وحدة واحدة يشارك النسخة الافتراضية singleton.

</details>

### تمرين متقدم

`PrismaModule` عامة ومستوردة في الجذر، واختبار الوحدة ما زال يتصل بقاعدة التطوير. ماذا غاب؟

<details>
<summary>الحل</summary>

الاختبار يستورد الوحدة الحقيقية. استبدل المزوّد بـ `overrideProvider(PrismaService).useValue(...)` أو لا تستورد الوحدة وابنِ `TestingModule` فيه الخدمة والمزيف فقط.

</details>

## أسئلة مراجعة

- لماذا التصدير جزء من الوحدة لا تفصيل؟
- متى يكون `@Global()` مبرَّرًا؟
- كم مجمع اتصالات نريد للتطبيق؟

## الخلاصة

`PrismaModule` تصدّر خدمة واحدة. الميزات تستوردها أو تراها بعد قرار عام واعٍ. الاستعلام يبقى في خدمة الميزة.

## ماذا بعد؟

[TypeORM كما في التوثيق الرسمي](./11-typeorm.md) يعرض البديل الذي يحافظ عليه Nest، بلا تغيير مسار Prisma.

## المصادر الرسمية

- NestJS — Prisma: https://docs.nestjs.com/data/prisma
- NestJS — Modules: https://docs.nestjs.com/modules
