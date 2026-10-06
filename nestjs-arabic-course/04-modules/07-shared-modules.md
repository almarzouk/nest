# Shared Modules

## الهدف من هذا الدرس

تعرف متى تخرج مزوّدًا يستخدمه أكثر من Feature Module إلى وحدة مشتركة، وكيف تصل إليه الوحدات عبر `imports` و `exports`.

## المشكلة التي يحلها

`BooksService` و `OrdersService` تحتاجان تسجيل وقت أو اتصالًا واحدًا. إن نسخت الصنف داخل كل وحدة صار عندك نسختان، وإن جعلت كل شيء في `AppModule` عادت الوحدة الجذر كيسًا بلا حدود.

## الفكرة ببساطة

الوحدة المشتركة تملك المزوّد وتصدّره. كل وحدة ميزة تستوردها حين تحتاجها. المشاركة قرار في `exports`، لا في نسخ الملف.

## التعريف التقني

Shared Module في Nest وحدة عادية: `providers` فيها ما تريد مشاركته، و `exports` يجعله مرئيًا لمن يضع الوحدة في `imports`. التوثيق يصف هذا النمط مع إعادة التصدير: وحدة وسيطة تستورد وحدة ثم تعيد تصديرها فيمر ما صدّرته إلى المستورد التالي.

لا يغيّر الاسم "Shared" سلوك الإطار. `@Module` لا يملك مفتاحًا اسمه shared. السلوك كله من `imports` و `exports`.

## Mental Model

```text
LoggerModule
  providers: [LoggerService]
  exports: [LoggerService]

BooksModule imports LoggerModule
OrdersModule imports LoggerModule
النسختان تحصلان على LoggerService نفسه ما دام التسجيل واحدًا
```

## أبسط مثال

```ts
import { Module } from "@nestjs/common";
import { LoggerService } from "./logger.service.js";

@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class LoggerModule {}
```

```ts
import { Module } from "@nestjs/common";
import { BooksController } from "./books.controller.js";
import { BooksService } from "./books.service.js";
import { LoggerModule } from "../logger/logger.module.js";

@Module({
  imports: [LoggerModule],
  controllers: [BooksController],
  providers: [BooksService],
})
export class BooksModule {}
```

## شرح المثال

`LoggerModule` تسجّل `LoggerService` وتصدّرها. `BooksModule` لا تعيد كتابة المزوّد في `providers`. الاستيراد يكفي حتى يحقن `BooksService` المسجّل.

`OrdersModule` تكرر الاستيراد نفسه. الحاوية ترى تسجيلًا واحدًا للصنف القادم من الوحدة المصدّرة، فتتشارك النسختان تلك النسخة في النطاق الافتراضي.

## ماذا يحدث داخليًا؟

عند الإقلاع يمشي Nest على `imports`. مزوّدات الوحدة المستورَدة التي ظهرت في `exports` تُضاف إلى سياق المستورد. المزوّد غير المصدَّر يبقى داخل وحدته حتى لو كان الملف قابلًا للاستيراد في TypeScript.

إعادة وضع `LoggerService` في `providers` عند `BooksModule` تسجيل جديد. النطاق الافتراضي singleton يكون لكل تسجيل، فتصير نسخة الكتب غير نسخة الطلبات.

## مثال عملي

في متجر صغير، `ClockModule` تصدّر ساعة واحدة تُستخدم في الطلبات والفواتير حتى لا يختلف "الآن" بين الميزتين أثناء الاختبار. الاختبار يستبدل الوحدة المستوردة، لا كل خدمة على حدة.

## أخطاء شائعة

- تسمية المجلد `shared` ثم نسيان `exports`، فيفشل الحقن رغم أن الملف مستورَد في TypeScript.
- وضع كل مزوّدات التطبيق في وحدة واحدة اسمها Shared حتى تضيع حدود الميزات.
- إعادة تسجيل الصنف في كل Feature Module مع استيراد الوحدة، فتظن أنك تشارك وأنت تنشئ نسخًا.
- جعل الوحدة `@Global()` لأن ثلاث وحدات تحتاجها. الاستيراد الصريح أوضح، والعالمي يُشرح في درس لاحق ويبقى نادرًا.

## متى نستخدمه؟

عندما مزوّدًا واحدًا، بلا إعداد معقّد، تحتاجه أكثر من وحدة ميزة: سجل، ساعة، مساعد تنسيق. وعندما تريد أن يبقى إنشاء النسخة في مكان واحد.

## متى لا نستخدمه؟

عندما المزوّد خاص بميزة واحدة. `BooksService` تعيش في `BooksModule` وتُصدَّر فقط إذا طلبتها وحدة أخرى. وعندما تحتاج الوحدة خيارات عند الإقلاع، فذلك Dynamic Module لاحقًا، لا مجلد مشترك ساكن.

## مقارنة

| | نسخ الصنف في كل وحدة | Shared Module |
| --- | --- | --- |
| عدد النسخ | تسجيل لكل وحدة | تسجيل واحد إن لم تُعد التسجيل |
| الحدود | كل ميزة تملك نسختها | الوحدة المشتركة تملك النسخة وتصدّرها |
| الاستيراد | لا حاجة لاستيراد وحدة | `imports: [LoggerModule]` |

مقابل `@Global()`: المشترك تطلبه كل وحدة في `imports`. العالمي يوفّر ذلك الاستيراد، ودرسه التالي يبيّن ثمن الغموض.

## تمارين

### تمرين سهل

أي مفتاحين في `@Module` يجعلان `IdService` متاحًا لـ `BooksModule` من غير أن تعيد تسجيله؟

<details>
<summary>الحل</summary>

`providers: [IdService]` و `exports: [IdService]` في الوحدة المالكة، ثم `imports` لتلك الوحدة داخل `BooksModule`.

</details>

### تمرين متوسط

`AuditService` مسجّل في `AuditModule` ومصدَّر. `BooksModule` تستورد `AuditModule` وتضع `AuditService` أيضًا في `providers`. كم نسخة في النطاق الافتراضي بين الوحدتين، ولماذا؟

<details>
<summary>الحل</summary>

تسجيلان، إذن نسختان. استيراد الوحدة يشارك تسجيل `AuditModule`. إعادة الصنف في `providers` تسجيل إضافي داخل `BooksModule`.

</details>

### تمرين متقدم

صمم وحدات لمتجر فيه كتب وطلبات. الاثنان يكتبان سطرًا في `AuditService`، و`OrdersService` يقرأ `BooksService` ليتأكد أن الكتاب موجود. ما الذي يُصدَّر، وما الذي يبقى خاصًا؟

<details>
<summary>الحل</summary>

`AuditModule` تصدّر `AuditService` ويستوردها الاثنان. `BooksModule` تصدّر `BooksService` لأن الطلبات تحتاجه. `OrdersService` و `BooksController` لا يُصدَّران إن لم تحتجهما وحدة أخرى. `AppModule` تستورد `BooksModule` و `OrdersModule` فقط.

</details>

## أسئلة مراجعة

- ما الذي يجعل وحدة "مشتركة" في Nest: اسم المجلد أم `exports`؟
- لماذا تفشل مشاركة النسخة إذا أعادت كل ميزة تسجيل الصنف؟
- متى يبقى المزوّد داخل وحدة الميزة ولا ينتقل إلى وحدة مشتركة؟

## الخلاصة

الوحدة المشتركة وحدة `@Module` تصدّر ما يتكرر استعماله. الوحدات الأخرى تستوردها. المشاركة تسجيل واحد ظاهر عبر `exports`، لا نسخ ملفات ولا جعل كل شيء عالميًا.

## ماذا بعد؟

الدرس التالي يسمي القاعدة التي رأيتها هنا: التغليف. ما لا يُصدَّر لا يُحقن من الخارج.

## المصادر الرسمية

- [Modules](https://docs.nestjs.com/modules)
- [Providers](https://docs.nestjs.com/providers)
