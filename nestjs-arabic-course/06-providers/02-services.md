# Service

## الهدف من هذا الدرس

وضع قاعدة العمل في Service يستدعيها Controller، بدلًا من كتابة القرار داخل المسار.

## المشكلة التي يحلها

غرامة التأخير إن بقيت داخل `@Post` تتكرر عند مسار الإعارة ومسار التقرير. Service تجمع القاعدة في صنف واحد يُختبر من غير HTTP.

## الفكرة ببساطة

Service هي Provider يحمل منطق التطبيق: حفظ واسترجاع وحساب. Controller يستقبل الطلب ويفوّض. التوثيق يبدأ بـ `CatsService` لهذا السبب، والأمر `nest g service fees` ينشئ الملف.

## التعريف التقني

Service صنف عليه `@Injectable()` وفيه حالة أو دوال. المثال الرسمي يحتفظ بمصفوفة خاصة ويعرض `create` و `findAll`. الحالة على النسخة تُشارك بين الطلبات لأن النطاق الافتراضي Singleton. هذا آمن في Node ما دامت الحالة حالة تطبيق، مثل كتالوج في الذاكرة، لا حالة زائر واحد.

لا تجعل Service تقرأ `@Body` أو `@Param`. المدخلات تصلها وسائط عادية من Controller.

## Mental Model

```text
FeesController.create(dto)
  -> FeesService.quote(dto.days)
  -> رقم
  -> Controller يعيد الرقم
  -> Nest ترسل الاستجابة
```

## أبسط مثال

```ts
import { Injectable } from '@nestjs/common';

export interface FeeQuote {
  days: number;
  amount: number;
}

@Injectable()
export class FeesService {
  private readonly quotes: FeeQuote[] = [];

  quote(days: number): FeeQuote {
    const row = { days, amount: days * 2 };
    this.quotes.push(row);
    return row;
  }

  findAll(): FeeQuote[] {
    return this.quotes;
  }
}
```

## شرح المثال

- `quotes` خاصة. خارج الصنف لا أحد يدفع عنصرًا إلى المصفوفة إلا عبر `quote`.
- `FeeQuote` واجهة وصف. استيرادها لاحقًا يكون `import type` لأنها ليست رمز حقن.
- `@Injectable()` يعلن أن الحاوية تدير الصنف عند تسجيله.
- `findAll` يعيد المصفوفة نفسها المخزنة. في مثال الذاكرة هذا متعمد. مستودع حقيقي يعيد نسخة أو نتيجة استعلام حتى لا يعدّل المستدعي التخزين مباشرة.

## ماذا يحدث داخليًا؟

نسخة واحدة من `FeesService` تخدم كل المعالجات التي حقنت الرمز، ضمن Module التسجيل أو من استوردها عبر `exports`. `quote` مرتين من طلبين مختلفين تدفع عنصرين في المصفوفة نفسها. هذا نتيجة Singleton، لا نتيجة خطأ في `@Post`. إن احتجت حالة لكل طلب، Scope `REQUEST` في الدرس الخاص به، وهو مكلف ويعدي من يحقن الخدمة.

Controller:

```ts
import { Body, Controller, Get, Post } from '@nestjs/common';
import { FeesService, type FeeQuote } from './fees.service.js';

@Controller('fees')
export class FeesController {
  constructor(private readonly fees: FeesService) {}

  @Post('quote')
  quote(@Body('days') days: number): FeeQuote {
    return this.fees.quote(days);
  }

  @Get()
  findAll() {
    return this.fees.findAll();
  }
}
```

`POST /fees/quote` رمزه 201. الجسم رقم أيام. الخدمة لا تعرف أن الرقم جاء من `@Body`.

## مثال عملي

قاعدة من طبقتين: الأيام المجانية ثم سعر اليوم.

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class FeesService {
  private readonly freeDays = 3;
  private readonly daily = 2;

  quote(days: number) {
    const billable = Math.max(days - this.freeDays, 0);
    return { days, amount: billable * this.daily };
  }
}
```

اختبار الخدمة يستدعي `quote(2)` ويتوقع `amount: 0` من غير تشغيل خادم. Controller يبقى غلاف HTTP حول هذه الدالة.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| حساب الغرامة داخل Controller وفي تقرير آخر | النسبتان تتباعدان عند تغيير اليوم المجاني |
| قراءة `request` داخل Service | الخدمة لا تُختبر إلا بطلب كامل |
| اعتبار المصفوفة الخاصة قاعدة بيانات دائمة | إعادة التشغيل تمسحها؛ المثال للذاكرة فقط |
| إرجاع `this.quotes` ثم تعديلها في Controller | المستدعي يغيّر حالة Singleton المشتركة |

## متى نستخدمه؟

عندما تتكرر قاعدة أو تحتاج اختبارًا بلا HTTP: تسعير، إتاحة نسخة، فتح إعارة، تجميع تقرير.

## متى لا نستخدمه؟

لترجمة `@Param` فقط من غير قاعدة. إن كانت الدالة تمرر الوسيط كما هو إلى مستودع، قد يكفي المستودع كـ Provider. Service تظهر عندما يوجد قرار.

## مقارنة

| Controller | Service |
| --- | --- |
| Decorators الطرق والمدخلات | دوال بأسماء المجال |
| تعيد قيمة إلى Nest | تعيد نتيجة القاعدة إلى المستدعي |
| مسجّل في `controllers` | مسجّل في `providers` |

## تمارين

### تمرين سهل

أين تعيش قاعدة «أول ثلاثة أيام بلا غرامة»؟

<details><summary>الحل</summary>

في `FeesService.quote`، لا في جسم `@Post`.

</details>

### تمرين متوسط

لماذا `findAll` في مثال المصفوفة يعيد العناصر التي أضافها طلب سابق؟

<details><summary>الحل</summary>

النسخة Singleton واحدة، والمصفوفة حقل عليها. الطلبات تشارك حالة التطبيق هذه. حالة زائر واحد لا تُخزَّن بهذا الشكل.

</details>

### تمرين متقدم

متى يكون مستودع الإعارات Provider أنسب من حشر SQL داخل `FeesService`؟

<details><summary>الحل</summary>

عندما التسعير يحتاج بيانات الإعارة لكن جملة SQL تفصيل تخزين. المستودع Provider يُحقن في الخدمة، والخدمة تبقى على قاعدة الأيام والسعر.

</details>

## أسئلة مراجعة

- ماذا يفوّض Controller إلى Service؟
- لماذا تُختبر `quote` من غير خادم؟
- ماذا يعني وجود حالة على نسخة Service الافتراضية؟

## الخلاصة

Service هي Provider لقاعدة العمل. Controller يمرر مدخلات ظاهرة ويستلم نتيجة، والحالة على النسخة الافتراضية حالة تطبيق مشتركة.

## ماذا بعد؟

ماذا يضيف `@Injectable` حتى تدير الحاوية هذا الصنف.

## المصادر الرسمية

- NestJS — Providers (Services): https://docs.nestjs.com/providers#services
