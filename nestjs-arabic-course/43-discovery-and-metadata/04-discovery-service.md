# DiscoveryService

## الهدف من هذا الدرس

العثور وقت التشغيل على المزودات والمتحكمات المسجّلة، وتصفيتها بمزخرف، لبناء سلوك إضافي من غير قائمة يدوية تتأخر عن الكود.

## المشكلة التي يحلها

حارس يعرف أهدافه لأنها على المسار. مهمة إقلاع تريد كل الخدمات المعلّمة بـ `@FeatureFlag('experimental')` حتى تسجلها في لوحة. كتابة أسمائها في مصفوفة تنسى الصنف الجديد. `DiscoveryService` من `@nestjs/core` يعدد ما سجله الإطار، و `DiscoveryModule` يجعل الخدمة قابلة للحقن.

## الفكرة ببساطة

استورد `DiscoveryModule`. احقن `DiscoveryService`. `getProviders()` و `getControllers()` تعيدان أغلفة `InstanceWrapper` فيها النسخة والرمز و `metatype`. صفِّ بالغلاف الذي يقرأ مزخرفًا صنعته بـ `DiscoveryService.createDecorator()`. مرر `metadataKey` ليجلب المعلَّمين فقط، أو `include` ليحصر المسح بوحدات.

## التعريف التقني

```ts
import { Module } from '@nestjs/common';
import { DiscoveryModule } from '@nestjs/core';
import { FlagScanner } from './flag.scanner.js';

@Module({
  imports: [DiscoveryModule],
  providers: [FlagScanner],
})
export class FlagsModule {}
```

المزخرف:

```ts
import { DiscoveryService } from '@nestjs/core';

export const FeatureFlag = DiscoveryService.createDecorator();
```

```ts
import { Injectable } from '@nestjs/common';
import { FeatureFlag } from './feature-flag.decorator.js';

@Injectable()
@FeatureFlag('experimental')
export class BetaExportService {}
```

المسح:

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { DiscoveryService } from '@nestjs/core';
import { FeatureFlag } from './feature-flag.decorator.js';

@Injectable()
export class FlagScanner implements OnModuleInit {
  constructor(private readonly discovery: DiscoveryService) {}

  onModuleInit(): void {
    const tagged = this.discovery.getProviders({
      metadataKey: FeatureFlag.KEY,
    });
    for (const wrapper of tagged) {
      const flag = this.discovery.getMetadataByDecorator(FeatureFlag, wrapper);
      void flag;
      void wrapper.metatype;
    }
  }
}
```

`getMetadataByDecorator(decorator, wrapper)` يقرأ قيمة الصنف. لقراءة دالة مرر اسمها وسيطًا ثالثًا. `include: [BillingModule]` يضيّق المسح.

`getProviders()` بلا خيار يعيد الجميع، بمن فيهم الداخليون. صفِّ قبل أن تنادي دوالًا على `instance`. قد يكون `instance` غير جاهز لبعض الأغلفة؛ تحقق قبل النداء.

هذا ليس بديل `Reflector` داخل الطلب. الحارس يعرف الدالة من `ExecutionContext` ولا يحتاج مسح التطبيق. الاكتشاف لمهمة ترى الكل: إضافات، تسجيل، فحص إقلاع.

## Mental Model

```text
DiscoveryModule
  -> getProviders({ metadataKey })
  -> أغلفة معلّمة
  -> اقرأ القيمة وتصرف مرة عند الإقلاع
```

المتحكمات لها `getControllers` بالشكل نفسه.

## أبسط مثال

`FlagScanner` أعلاه. بعد الإقلاع تحتوي `tagged` على `BetaExportService` إن كانت مسجّلة في وحدة مستوردة في الرسم. مزود غير مسجّل في أي `providers` لن يظهر ولو لصقت المزخرف على الملف.

## شرح المثال

`createDecorator` يسجل الصنف عند استعمال المزخرف، لذلك `metadataKey: FeatureFlag.KEY` يكفي بدل تصفية الكل يدويًا. القيمة `'experimental'` تُقرأ بالمساعد. ضع الماسح في `onModuleInit` لا في كل طلب. المسح المتكرر لكل `GET` يعيد عمل الإقلاع على المسار الساخن.

## ماذا يحدث داخليًا؟

1. `DiscoveryModule` يصدّر الخدمة.
2. عند التهيئة يستطيع الماسح رؤية الأغلفة المسجّلة حتى تلك اللحظة. ضعه حيث اكتمل استيراد الوحدات التي تهمك، أو امسح في `onApplicationBootstrap` إن احتجت الرسم كاملًا.
3. `metadataKey` يحد النتائج بمن لصق المزخرف.
4. `wrapper.metatype` الصنف، و `wrapper.instance` النسخة إن بُنيت.
5. `include` يقص الوحدات.
6. وحدة كسولة لم تُحمَّل بعد ليست في القائمة حتى `load`.

## مثال عملي

تسجيل مقاييس: `@Measured()` على دوال خدمة عبر المزخرف، ثم في `onApplicationBootstrap` تمشي الأغلفة وتقرأ metadata الدالة بالوسيط الثالث لتعرف أسماء العمليات. لا تستدعِ الدالة أثناء المسح. خزّن الأسماء في مسجّل `DEFAULT`. مسار الطلب يزيد العداد في المعترض عبر `Reflector` على `getHandler()`، والاكتشاف هنا فقط لبناء الفهرس مرة.

## أخطاء شائعة

- نسيان `DiscoveryModule` في `imports`.
- مسح في كل طلب.
- توقع ظهور صنف غير مسجّل كمزود.
- نداء `instance` وهو فارغ على غلاف لم يُبنَ.
- استعمال الاكتشاف بدل حارس لقراءة `@Public()` على المسار الحالي. السياق يكفي.

## متى نستخدمه؟

فهرسة ما لُصق عند الإقلاع: أعلام، مستمعون، عمليات قياس.

## متى لا نستخدمه؟

قرار مسار واحد. `Reflector` مع `ExecutionContext` أوضح وأرخص.

## مقارنة

| الأداة | متى | المدخل |
| --- | --- | --- |
| `Reflector` | أثناء الطلب على هدف تعرفه | `getHandler` / `getClass` |
| `DiscoveryService` | مسح المسجّلين | `getProviders` / `getControllers` |
| قائمة يدوية | عندما الأسماء عقد خارجي ثابت | مصفوفة تحدّثها أنت |

## تمارين

### تمرين سهل

أي وحدة تستوردها قبل حقن `DiscoveryService`؟

<details>
<summary>الحل</summary>

`DiscoveryModule` من `@nestjs/core`.

</details>

### تمرين متوسط

ماذا يضيف `metadataKey: FeatureFlag.KEY` على `getProviders()`؟

<details>
<summary>الحل</summary>

يحصر النتائج في الأصناف التي لصق عليها المزخرف المولَّد من `DiscoveryService.createDecorator`.

</details>

### تمرين متقدم

لماذا قد تنقل المسح من `onModuleInit` إلى `onApplicationBootstrap`؟

<details>
<summary>الحل</summary>

لأن الثاني بعد تهيئة كل الوحدات. ماسح في ورقة قد لا يرى مزودًا في وحدة تُهيأ أقرب إلى الجذر إن اعتمدت على ترتيب لم يكتمل بعد في حدود مسحك.

</details>

## أسئلة مراجعة

- ماذا يحمل `InstanceWrapper` الذي يهمك؟
- كيف تقرأ metadata على دالة لا على الصنف؟
- لماذا المسح ليس مكان `@Public()`؟

## الخلاصة

`DiscoveryService` يعدد المسجّل ويصفّي بمزخرف الاكتشاف. استورده عبر `DiscoveryModule`، وافهرس عند الإقلاع، واترك قرار الطلب لـ `Reflector`.

## ماذا بعد؟

`ExecutionContext` بعمق: نوع القناة وكيف تصل إلى الطلب أو رسالة RPC أو عميل المقبس.

## المصادر الرسمية

- NestJS — Discovery service: https://docs.nestjs.com/fundamentals/discovery-service
