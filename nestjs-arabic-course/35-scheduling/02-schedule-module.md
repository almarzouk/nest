# @nestjs/schedule

## الهدف من هذا الدرس

تفعيل الجدولة في التطبيق مرة واحدة عبر `ScheduleModule.forRoot`، وربط الدوال المزينة كمزودات.

## المشكلة التي يحلها

كتابة `@Cron` على صنف غير مسجّل لا توقظ شيئًا. واستيراد الوحدة في كل ميزة مرتين قد يكرر التسجيل. المؤقتات تحتاج أن تبدأ مع الإقلاع من وحدة واحدة تعرفها Nest.

## الفكرة ببساطة

`npm i @nestjs/schedule` ثم `ScheduleModule.forRoot()` في الجذر أو في وحدة مستوردة مرة. الدوال تعيش على `@Injectable()` مذكور في `providers`. الزينة `@Cron` و `@Interval` و `@Timeout` من الحزمة نفسها. بلا `forRoot` الزينة صامتة.

## التعريف التقني

```ts
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';
import { CleanupService } from './cleanup.service.js';

@Module({
  imports: [ScheduleModule.forRoot()],
  providers: [CleanupService],
})
export class AppModule {}
```

`forRoot` بلا خيارات يكفي للبدء. الصنف:

```ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class CleanupService {
  @Cron(CronExpression.EVERY_HOUR)
  async dropOldTempFiles() {
    await this.files.dropOlderThan('var/tmp', 24 * 60 * 60 * 1000);
  }
}
```

`CronExpression.EVERY_HOUR` ثابت مقروء بدل نمط تصعب مراجعته. تفاصيل النمط في الدرس التالي. هنا المطلوب أن الصنف مزوّد والوحدة مستوردة.

لا تضع `forRoot` في `CoversModule` و `MailModule` معًا إن كان الاثنان يُستوردان دائمًا، حتى لا تتتبع أين بدأت الجدولة. مكان واحد في الجذر أوضح.

## Mental Model

```text
الإقلاع
  ScheduleModule.forRoot
    يقرأ @Cron على المزودات
      مؤقت لكل دالة
```

## أبسط مثال

خدمة بلا حقن للطابور عندما يكون العمل مسح ملفات محلية قصيرة:

```ts
@Cron(CronExpression.EVERY_HOUR)
async dropOldTempFiles() {
  await this.files.dropOlderThan('var/tmp', 86_400_000);
}
```

`files` خدمة قرص تُحقن في الباني. الاختبار يستدعي `dropOldTempFiles()` مباشرة ولا ينتظر الساعة.

## شرح المثال

الزينة تسجل الدالة عند إنشاء المزود. الاسم `dropOldTempFiles` يظهر في السجلات إن أضفت سجلًا في أول السطر. العمل نفسه يبقى في `files` حتى تستطيع اختبار الحذف بمعيار العمر من غير Nest. الساعة ليست جزءًا من صحة «الملف أقدم من يوم».

## ماذا يحدث داخليًا؟

عند الإقلاع تمسح Nest المزودات بحثًا عن البيانات الوصفية للجدولة وتبدأ المؤقتات في العملية. إيقاف العملية يوقف المؤقتات. لا توجد قائمة في Redis من هذا التسجيل وحده. إن أعدت التشغيل في منتصف الساعة، النبضة التالية حسب النمط لا «تعوّض» النبضة التي فاتت إلا إذا كتبت ذلك. لا تفترض أن `@Cron` اللحظي سيعيد تشغيل ما سقط.

الدالة `async` التي ترمي ترفض الوعد. المكتبة لا تعيد المحاولة كـ BullMQ. سجّل الخطأ داخل الدالة أو دع مرشحًا يراه، ولا تترك رفضًا صامتًا إن كان التنظيف حرجًا. إن استمر العمل أطول من الفترة، قد تتداخل نبضتان. امنع التداخل بعلم `running` في الخدمة أو بالإدراج إلى طابور.

## مثال عملي

الجذر يستورد الجدولة والميزات:

```ts
@Module({
  imports: [ScheduleModule.forRoot(), CoversModule, BooksModule],
  providers: [CleanupService],
})
export class AppModule {}
```

`CoversModule` لا تستورد `ScheduleModule` مرة ثانية. إن احتاجت تذكير المسودات، ضع `@Cron` على مزوّد في وحدة تُحمَّل مع الجذر، أو انقل المزود إلى الجذر حتى ترى التسجيل في ملف واحد أثناء التعلم.

## أخطاء شائعة

- `@Cron` على صنف في `controllers` أو على صنف غير مذكور في `providers`.
- `forRoot` في كل وحدة ميزة.
- توقع إعادة المحاولة من الحزمة بعد الرمي.
- انتظار الساعة في اختبار الوحدة.

## متى نستخدمه؟

عندما يظهر أول عمل زمني حقيقي: تنظيف أو تقرير. التسجيل مرة في جذر التطبيق.

## متى لا نستخدمه؟

قبل وجود دالة تستحق الوقت. `forRoot` وحده لا ينظف شيئًا. ولا تستورده لحل إعادة محاولة القص.

## مقارنة

| الشرط | الأثر |
| --- | --- |
| `forRoot` + مزوّد مزين | المؤقت يعمل |
| زينة بلا تسجيل المزود | لا شيء |
| `forRoot` مرتين في شجرتين | تتبع أصعب وتكرار محتمل |
| رمي من الدالة | لا `attempts` تلقائية |

## تمارين

### تمرين سهل

أي استدعاء يفعّل زينة الجدولة؟

<details>
<summary>الحل</summary>

`ScheduleModule.forRoot()` من `@nestjs/schedule`.

</details>

### تمرين متوسط

كيف تختبر الحذف دون انتظار ساعة؟

<details>
<summary>الحل</summary>

استدعِ الدالة أو `files.dropOlderThan` مباشرة في الاختبار. الزينة ليست موضوع الاختبار.

</details>

### تمرين متقدم

لماذا التداخل خطر إن طال التنظيف أكثر من الفترة؟

<details>
<summary>الحل</summary>

النبضة التالية تبدأ بينما الأولى ما زالت تمشي على المجلد نفسه. علم تشغيل أو نقل العمل إلى طابور يمنع مسارين متوازيين.

</details>

## أسئلة مراجعة

- أين يوضع `forRoot`؟
- هل فشل الدالة يُعاد كسياسة طابور؟
- ماذا يختبر الاختبار: النمط أم الحذف؟

## الخلاصة

`ScheduleModule.forRoot` مرة، والمزوّد المسجّل يحمل الزينة. الحذف يُختبر بنداء مباشر، والتداخل والفشل مسؤوليتك لا سياسة `attempts`.

## ماذا بعد؟

نمط `@Cron` الذي يشغّل التنظيف في ساعة معروفة لا كل 60 ثانية فقط.

## المصادر الرسمية

- NestJS — Task scheduling: https://docs.nestjs.com/techniques/task-scheduling
