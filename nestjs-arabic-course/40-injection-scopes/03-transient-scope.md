# TRANSIENT

## الهدف من هذا الدرس

تمييز النسخة الخاصة بكل مكان حقن عن نسخة الطلب، ومعرفة أن هذا النطاق لا يصعد إلى المستهلك.

## المشكلة التي يحلها

مسجّل يريد أن يعرف من استدعاه، أو مولّد حالة يجب ألا يُشارك بين خدمتين حتى داخل الطلب نفسه. `REQUEST` ما زال يشارك النسخة داخل شجرة ذلك الطلب، ويصعد بالنطاق. `TRANSIENT` يعطي كل مستهلك كائنًا مخصصًا ويترك المستهلك على نطاقه.

## الفكرة ببساطة

المزود العابر غير مشترك بين المستهلكين. إن حقنت `LoggerService` العابرة في `BooksService` و `OrdersService` فلكل واحدة نسخة. `BooksService` إن كانت `DEFAULT` تبقى واحدة، ونسخة المسجّل تعيش داخلها. حقن `BooksService` في مكان آخر لا يصنع `BooksService` جديدة.

## التعريف التقني

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class LoggerService {}
```

وعلى المزود المخصص: `scope: Scope.TRANSIENT` بجانب `provide` و `useClass`.

التوثيق: إن حقن `DogsService` الـ Singleton مزودًا عابرًا، يأخذ نسخة طازجة، ويبقى `DogsService` نفسه Singleton. من يحقن `DogsService` لا يحصل على نسخة جديدة منه. إن أردت ذلك علّم `DogsService` صراحة `TRANSIENT`.

رمز `INQUIRER` من `@nestjs/core` يعطي الصنف الذي بُني المزود داخله. مفيد مع النطاق العابر للسجلات والمقاييس:

```ts
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private readonly parentClass: object) {}

  sayHello(message: string): void {
    const name = this.parentClass?.constructor?.name ?? 'unknown';
    console.log(`${name}: ${message}`);
  }
}
```

`get()` على `ModuleRef` لا يسترجع المزودات ذات النطاق، بما فيها العابرة. استخدم `resolve()` كما في فصل `ModuleRef`.

## Mental Model

```text
BooksService (واحدة)  -> HelloService نسخة أ
OrdersService (واحدة) -> HelloService نسخة ب
نسخة أ !== نسخة ب
BooksService تبقى واحدة مهما كثرت الطلبات
```

لا context id لكل طلب بسبب هذا المزود وحده.

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service.js';

@Injectable()
export class AppService {
  constructor(private readonly helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');
    return 'Hello world!';
  }
}
```

عند النداء تطبع `AppService: My name is getRoot` لأن `INQUIRER` هو الصنف الأب الذي حقن النسخة العابرة.

## شرح المثال

`HelloService` عابرة، فالنسخة التي داخل `AppService` ليست التي داخل خدمة أخرى. `INQUIRER` يُحقن وقت البناء ويعرف الأب. لو كانت الخدمة `DEFAULT` لاشترك الجميع في مسجّل واحد ولما اختلف الأب بشكل مفيد لكل مستهلك.

## ماذا يحدث داخليًا؟

1. الحاوية ترى الاعتماد العابر أثناء بناء `AppService`.
2. تنشئ `HelloService` مخصصة لهذا البناء وتحقن `INQUIRER`.
3. تخزن `AppService` كـ Singleton مع حقل يشير إلى تلك النسخة.
4. بناء `OrdersService` ينشئ `HelloService` ثانية.
5. طلبات لاحقة تعيد `AppService` نفسها ونسخة المسجّل نفسها المرتبطة بها.
6. لا عدوى: وجود العابر لا يعلّم الأب `REQUEST` ولا `TRANSIENT`.

## مثال عملي

ساعة توقيت لكل خدمة تقيس عملها من غير سباق على حقول مشتركة:

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class Stopwatch {
  private started = 0;

  start(): void {
    this.started = Date.now();
  }

  millis(): number {
    return Date.now() - this.started;
  }
}
```

`ImportService` و `ExportService` تحقنان `Stopwatch`. استدعاء `start` في الاستيراد لا يصفّر ساعة التصدير. لو كانت الساعة `DEFAULT` لشاركت الحقل `started` كل الخدمات. لو كانت `REQUEST` لصعدت الخدمتان إلى نطاق الطلب إن حقنتاها، ودفعت ثمن الشجرة لكل طلب.

## أخطاء شائعة

- انتظار عدوى مثل `REQUEST`. الأب يبقى على نطاقه.
- استخدام `TRANSIENT` لعميل قاعدة، فتفتح عميلًا لكل خدمة من غير داعٍ.
- استرجاع النسخة بـ `ModuleRef.get()`. العابر يُحل بـ `resolve()`.
- نسيان `INQUIRER` ثم محاولة تخزين اسم الأب يدويًا على Singleton.
- تعليم المتحكم `TRANSIENT` لحل مشكلة مستخدم. المتحكم العابر ليس بديل الديكور.

## متى نستخدمه؟

حالة صغيرة يجب أن تُعزل لكل مستهلك: مسجّل يعرف أباه، ساعة، سياق بناء. الأب يبقى Singleton.

## متى لا نستخدمه؟

لمشاركة اتصال، أو لعزل كل طلب. العزل لكل طلب هو `REQUEST` إن كانت النسخة نفسها يجب أن تختلف مع الطلب. المستخدم الحالي يبقى على الطلب لا على النطاق.

## مقارنة

| | `DEFAULT` | `REQUEST` | `TRANSIENT` |
| --- | --- | --- | --- |
| المشاركة | كل التطبيق | داخل الطلب | لا تُشارك بين المستهلكين |
| عدوى | — | تصعد | لا تصعد |
| `INQUIRER` | أب واحد غير مفيد | حسب الشجرة | الأب الذي حقن هذه النسخة |

## تمارين

### تمرين سهل

خدمتان Singleton تحقن `LoggerService` العابرة. كم نسخة مسجّل؟

<details>
<summary>الحل</summary>

نسختان، واحدة لكل مستهلك. الخدمتان تبقيان Singleton.

</details>

### تمرين متوسط

ماذا يطبع `INQUIRER` في المثال الرسمي داخل `AppService`؟

<details>
<summary>الحل</summary>

اسم الأب `AppService`، فالسطر يبدأ بـ `AppService:`.

</details>

### تمرين متقدم

لماذا إبقاء `Stopwatch` عابرة أرخص من تعليم سلسلة الكتب `REQUEST` لأجل ساعة؟

<details>
<summary>الحل</summary>

العابر يبني كائن ساعة صغير لكل مستهلك عند الإقلاع ويبقي المتحكم Singleton. `REQUEST` يعيد بناء المتحكم وكل السلسلة الصاعدة لكل طلب.

</details>

## أسئلة مراجعة

- هل النطاق العابر يعدي؟
- متى يفيد `INQUIRER`؟
- كيف تسترجع نسخة عابرة من `ModuleRef`؟

## الخلاصة

`TRANSIENT` نسخة مخصصة لكل حقن، والمستهلك يبقى على نطاقه. استخدمه لعزل حالة صغيرة، لا لاتصال مشترك ولا للمستخدم الحالي.

## ماذا بعد؟

الأثر على الأداء: ماذا يكلف صعود `REQUEST` في الشجرة.

## المصادر الرسمية

- NestJS — Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
