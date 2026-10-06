# خريطة دورة الحياة

## الهدف من هذا الدرس

رؤية المراحل الثلاث لتطبيق Nest من الإقلاع حتى خروج العملية، ومعرفة أي خطاف يقع في أي مرحلة قبل كتابة أي اتصال أو تنظيف.

## المشكلة التي يحلها

وضع `$connect` داخل `constructor`، أو إغلاق القاعدة بعد أن أُغلقت المقابس، أو انتظار إشارة إيقاف لن تصل أصلًا. الخريطة تمنع هذه الأخطاء لأنها تُظهر أن الإقلاع والتشغيل والإيقاف ثلاث مراحل، وأن خطافات الإيقاف لا تُستدعى وحدها.

## الفكرة ببساطة

Nest يدير عمر التطبيق وعناصره. أنت تسجّل دوالًا على Module أو Provider أو Controller، والإطار يستدعيها في أحداث معروفة. الإقلاع خطافان: `onModuleInit` ثم `onApplicationBootstrap`. التشغيل هو الاستماع للاتصالات. الإيقاف ثلاثة خطافات، وبالترتيب: `onModuleDestroy` ثم `beforeApplicationShutdown` ثم `onApplicationShutdown`.

## التعريف التقني

صفحة Lifecycle events تقسم العمر إلى initializing و running و terminating. الخطافان الأولان يُستدعيان عند `app.init()` أو `app.listen()` أو عند `NestFactory.createApplicationContext()` الذي يهيّئ التطبيق بنفسه.

خطافات الإيقاف تُستدعى عند `app.close()` صراحة، أو عند إشارة نظام مثل `SIGTERM` بعد `enableShutdownHooks()`. من غير أحد هذين المسارين تبقى دوال الإيقاف صامتة.

الواجهات: `OnModuleInit` و `OnApplicationBootstrap` و `OnModuleDestroy` و `BeforeApplicationShutdown` و `OnApplicationShutdown`. الواجهة اختيارية بعد الترجمة لأن TypeScript يمحوها، وتبقى مفيدة للأنواع.

| الخطاف | متى |
| --- | --- |
| `onModuleInit()` | بعد حل اعتماديات الوحدة المضيفة |
| `onApplicationBootstrap()` | بعد تهيئة كل الوحدات، وقبل الاستماع |
| `onModuleDestroy()` | بعد إشارة الإنهاء |
| `beforeApplicationShutdown(signal)` | بعد اكتمال كل `onModuleDestroy`، ثم تُغلق الاتصالات |
| `onApplicationShutdown(signal)` | بعد إغلاق الاتصالات، آخر خطوة في `app.close()` |

Nest يستدعي خطافات الإقلاع من الوحدات الأبعد عن الجذر (والوحدات العامة) نحو `AppModule`، وينتظر كل وحدة قبل التي تليها. خطافات الإيقاف تعكس هذا الاتجاه: من الجذر نحو الأوراق.

## Mental Model

```text
initializing
  أوراق الرسم -> onModuleInit -> onApplicationBootstrap -> الجذر
running
  listen / الاتصالات مفتوحة
terminating   (app.close أو إشارة بعد enableShutdownHooks)
  الجذر -> onModuleDestroy -> beforeApplicationShutdown
  -> إغلاق الاتصالات -> onApplicationShutdown -> الأوراق
```

أصناف `Scope.REQUEST` خارج هذه الخريطة. عمرها مربوط بالطلب، وخطافات التطبيق لا تُستدعى عليها.

## أبسط مثال

```ts
import { Injectable, OnModuleInit, OnApplicationBootstrap } from '@nestjs/common';

@Injectable()
export class BootLogService implements OnModuleInit, OnApplicationBootstrap {
  onModuleInit(): void {
    console.log('init');
  }

  onApplicationBootstrap(): void {
    console.log('bootstrap');
  }
}
```

على تطبيق فيه هذه الخدمة فقط، الترتيب في الطرفية هو `init` ثم `bootstrap` ثم فتح المنفذ. لا يظهر سطر إيقاف إلا إذا استدعيت `app.close()` أو فعّلت مستمعي الإشارات.

## شرح المثال

الصنف Provider عادي. الواجهتان تفرضان اسم الدالة. Nest يكتشف الدالة بالاسم أثناء الإقلاع، فتنفيذ الواجهة يساعدك في المحرر ولا يُشترط كي يعمل الخطاف. `console.log` هنا أداة قراءة للترتيب، ويُحذف عندما تضع عملًا حقيقيًا.

## ماذا يحدث داخليًا؟

1. تحميل الملفات يلصق metadata.
2. الحاوية تبني الرسم وتحل الاعتماديات من الأوراق.
3. لكل وحدة، بالبعد عن الجذر: `onModuleInit` ثم لاحقًا `onApplicationBootstrap`. الوعد يُنتظر.
4. `listen` يبدأ قبول الاتصالات. هذه مرحلة running.
5. `app.close()` أو إشارة مفعّلة تبدأ terminating بالترتيب العكسي بين الوحدات، ثم إغلاق السيرفر، ثم `onApplicationShutdown`.
6. `app.close()` نفسه لا يقتل عملية Node إن بقي `setInterval` أو مهمة خلفية.

## مثال عملي

ارسم لمشروع الكتب ثلاث مراحل على ورقة قبل أن تختار ملفًا:

```text
PrismaModule.onModuleInit     -> $connect
OrdersModule.onApplicationBootstrap -> لا تفتح منفذًا هنا
إشارة SIGTERM
PrismaModule.onModuleDestroy  -> $disconnect
قبل إغلاق HTTP                -> beforeApplicationShutdown
بعد إغلاق المقابس             -> onApplicationShutdown
```

إن احتجت المستخدم الحالي فذلك طلب، وخطاف التطبيق ليس مكانه. الدرس التالي يفصل `OnModuleInit` وحده.

## أخطاء شائعة

- حفظ خريطة Request lifecycle (Middleware و Guard) على أنها خريطة إقلاع العملية.
- افتراض أن `SIGTERM` يستدعي الإيقاف من غير `enableShutdownHooks()`.
- وضع عمل طلب داخل خطاف إقلاع.
- توقع خطاف على خدمة `Scope.REQUEST`.
- الاعتماد على ترتيب البناء بين خدمتين في الوحدة نفسها من غير انتظار صريح.

## متى نستخدمه؟

عند أي سؤال: أين أفتح عميلًا، وأين أغلقه، وهل الكود يعمل قبل `listen` أم بعده.

## متى لا نستخدمه؟

اترك الخريطة عندما يكون العمل داخل طلب واحد: تحقق، صلاحية، أو قراءة مستخدم. تلك محطات الفصل 08.

## مقارنة

| المرحلة | سؤالها | خطافاتها |
| --- | --- | --- |
| initializing | هل الاعتماد جاهز قبل الباب؟ | `onModuleInit` ثم `onApplicationBootstrap` |
| running | هل نقبل عملًا؟ | لا خطاف تطبيق؛ الطلب يمر بمساره |
| terminating | هل نغلق بالترتيب؟ | destroy ثم before ثم shutdown |

## تمارين

### تمرين سهل

ما الخطافان اللذان يسبقان `listen`؟

<details>
<summary>الحل</summary>

`onModuleInit` ثم `onApplicationBootstrap`.

</details>

### تمرين متوسط

وحدة `DatabaseModule` مستوردة من `BooksModule` المستوردة من الجذر. أي وحدة ترى `onModuleInit` أولًا؟

<details>
<summary>الحل</summary>

الأبعد عن الجذر أولًا، فـ `DatabaseModule` قبل `BooksModule` قبل الجذر. Nest ينتظر كل وحدة.

</details>

### تمرين متقدم

لماذا تبقى خدمة الطلب بلا `onModuleDestroy` رغم أن العملية استلمت `SIGTERM`؟

<details>
<summary>الحل</summary>

أصناف `Scope.REQUEST` غير مربوطة بدورة حياة التطبيق. تُنشأ للطلب وتُجمع بعد الرد. التنظيف الذي يخص العملية يوضع على Singleton.

</details>

## أسئلة مراجعة

- ما المراحل الثلاث؟
- ما شرط استدعاء خطافات الإيقاف عند إشارة النظام؟
- أي اتجاه يسلكه الإيقاف بين الوحدات؟

## الخلاصة

الخريطة ثابتة: إقلاع من الأوراق، تشغيل، ثم إيقاف من الجذر إن أُغلق التطبيق أو وصلت إشارة بعد تفعيل المستمعين. الترتيب قرار الإطار.

## ماذا بعد؟

`OnModuleInit`: ماذا تضع فيه بعد حل اعتماديات الوحدة.

## المصادر الرسمية

- NestJS — Lifecycle events: https://docs.nestjs.com/fundamentals/lifecycle-events
