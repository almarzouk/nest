# Request Object

## الهدف من هذا الدرس

حقن كائن طلب المنصة بـ `@Req()` أو `@Request()`، ومعرفة متى يبقى `@Param` و `@Query` و `@Body` و `@Headers` أوضح.

## المشكلة التي يحلها

بعض الحقول لا Decorator مخصصًا لها في الاستعمال اليومي: `originalUrl` و `protocol` و `hostname` معًا. تمرير `req` كاملًا إلى كل المعالجات يعيد التطبيق إلى دالة Express التي تعرف كل شيء. `@Req()` متاح للحالة التي تحتاج الكائن، والـ Decorators المخصصة تبقى للمدخلات المعتادة.

## الفكرة ببساطة

`@Req()` يحقن طلب Express الافتراضي، أو طلب Fastify إن كان هو المحول. `@Request()` الاسم نفسه. للأنواع ثبّت `@types/express` واستورد النوع استيراد نوع فقط.

## التعريف التقني

التوثيق يمرر الطلب هكذا:

```ts
findAll(@Req() request: Request)
```

`import type { Request } from 'express'` استيراد نوع. هذا آمن هنا لأن `Request` واجهة وصف، و `@Req()` لا يستخدم النوع كمفتاح حقن. `import type` على صنف Service يكسر الحقن، وذلك في فصل Dependency Injection.

حقول شائعة على طلب Express: `params` و `query` و `headers` و `body` و `method` و `originalUrl` و `ip` و `protocol`. الحقول الأربعة الأولى لها Decorators. `@Ip()` يحقن `req.ip` وحده إن كان هو كل ما تحتاجه.

`@Req()` لا يبدّل أسلوب الاستجابة. الذي يبدّله `@Res()` و `@Next()`.

## Mental Model

```text
المعالج يحتاج originalUrl + method + ip
  -> @Req()

المعالج يحتاج معرف المسار فقط
  -> @Param('id')
```

## أبسط مثال

```ts
import { Controller, Get, Req } from '@nestjs/common';
import type { Request } from 'express';

@Controller('audits')
export class AuditsController {
  @Get('incoming')
  incoming(@Req() request: Request) {
    return {
      method: request.method,
      url: request.originalUrl,
      ip: request.ip,
    };
  }
}
```

## شرح المثال

- `@Req()` و `@Request()` مترادفان. استورد أحدهما من `@nestjs/common`.
- الكائن المعاد يختار ثلاثة حقول. بقية الطلب لا تخرج في JSON.
- `GET /audits/incoming?from=desk` يجعل `originalUrl` يتضمن المسار والاستعلام. `@Query('from')` كان يكفي لو لزم `from` فقط.
- الرمز 200 لأن الطريقة GET والقيمة كائن فيسلسلها Nest.

## ماذا يحدث داخليًا؟

Nest تمرر كائن المنصة الذي بنته للطلب. على Express هذا `req`. تغيير `request.query` داخل المعالج يغيّر الكائن نفسه الذي تراه أي طبقة لاحقة في الطلب نفسه. اقرأ الحقول وأعد قيمة جديدة.

`@Session()` و `@Cookies()` و `@SignedCookies()` و `@HostParam()` و `@Ip()` Decorators متخصصة موثّقة بجانب `@Req()`. استخدم المتخصص عندما يغطي حقلًا واحدًا. الكائن الكامل عندما تجمع حقول بروتوكول في أثر تدقيق واحد.

## مثال عملي

```ts
import { Controller, Get, Ip, Param, Req } from '@nestjs/common';
import type { Request } from 'express';
import { AuditsService } from './audits.service.js';

@Controller('audits')
export class ResourceAuditsController {
  constructor(private readonly audits: AuditsService) {}

  @Get('resources/:id')
  resource(
    @Param('id') id: string,
    @Ip() ip: string,
    @Req() request: Request,
  ) {
    return this.audits.trace({
      id,
      ip,
      protocol: request.protocol,
      url: request.originalUrl,
    });
  }
}
```

`id` من المسار، و `ip` من `@Ip()`، والبروتوكول والعنوان الكامل من الكائن. الخدمة تتسلم أثرًا جاهزًا وتخزنه. المعالج لا يكتب جدول التدقيق بنفسه.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `@Req() request` ثم `request.body.isbn` في كل المسارات | عقد الجسم يختفي داخل كائن واسع |
| `import { Request } from 'express'` ظنًا أنه مزوّد Nest | `Request` نوع Express، والحقن يتم بـ `@Req()` |
| تعديل `request.headers` أثناء المعالجة | الطبقات التالية في الطلب نفسه ترى التعديل |
| توقع أن `@Req()` وحده يوقف تسلسل القيمة المعادة | إيقاف الأسلوب القياسي يتم بـ `@Res()` أو `@Next()` |

## متى نستخدمه؟

لتجميع حقول بروتوكول لا يغطيها Decorator واحد: `originalUrl` مع `protocol` مع `method` في أثر تدقيق.

## متى لا نستخدمه؟

لقراءة `id` أو `page` أو JSON أو رأس واحد. `@Param` و `@Query` و `@Body` و `@Headers` تسمي المدخل في التوقيع. `@Ip()` يكفي لعنوان العميل وحده.

## مقارنة

| الأداة | ماذا يحقن |
| --- | --- |
| `@Req()` / `@Request()` | كائن طلب المنصة |
| `@Param` `@Query` `@Body` `@Headers` | جزء مسمّى من الطلب |
| `@Ip()` | `req.ip` فقط |

## تمارين

### تمرين سهل

ما الحزمة التي تعطي نوع `Request` على محول Express؟

<details><summary>الحل</summary>

`@types/express`، والاستيراد `import type { Request } from 'express'`.

</details>

### تمرين متوسط

المعالج يحتاج `page` فقط. أي توقيع أوضح من `@Req()`؟

<details><summary>الحل</summary>

`@Query('page') page?: string`. الاستعلام موثّق في الوسيط، وباقي `req` لا يدخل الدالة.

</details>

### تمرين متقدم

لماذا `import type` لـ `Request` مقبول هنا بينما `import type` لصنف Service يكسر الحقن؟

<details><summary>الحل</summary>

`@Req()` يحقن كائن المنصة من غير أن يبحث عن رمز `Request` في الحاوية. نوع Service في `constructor` هو مفتاح الحقن، و `import type` يمسح ذلك الرمز وقت التشغيل.

</details>

## أسئلة مراجعة

- ما الاسم المرادف لـ `@Req()`؟
- أي Decorator يحقن عنوان IP وحده؟
- هل `@Req()` يضع المعالج في وضع المكتبة؟

## الخلاصة

`@Req()` يفتح كائن طلب المنصة عندما تحتاج حقول بروتوكول مجتمعة. مدخلات المورد تبقى على Decorators الأجزاء، والأسلوب القياسي لإرجاع القيمة يبقى كما هو.

## ماذا بعد؟

كائن الاستجابة ووضع المكتبة: Response Object و Library-specific Mode.

## المصادر الرسمية

- NestJS — Request object: https://docs.nestjs.com/controllers#request-object
