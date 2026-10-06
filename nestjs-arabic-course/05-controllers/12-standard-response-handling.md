# Standard Nest Response Handling

## الهدف من هذا الدرس

ماذا يحدث للقيمة التي يعيدها المعالج عندما يبقى الإرسال عند Nest.

## المشكلة التي يحلها

بناء `res.status().json()` في كل مسار يكرر التسلسل والرمز. الأسلوب القياسي يعيد قيمة، وNest تختار JSON أو إرسالًا خامًا، وتنتظر `Promise`، وتشترك في `Observable` حتى آخر قيمة.

## الفكرة ببساطة

كائن أو مصفوفة تُسلسل JSON. النص والعدد والقيمة المنطقية تُرسل كما هي من غير محاولة تسلسل JSON. الرمز 200، و POST يستخدم 201، إلا إذا غيّره `@HttpCode`. `@Header` يضيف رأس استجابة مع بقاء هذا الأسلوب.

## التعريف التقني

التوثيق يسمّي هذا الخيار Standard. الشرط أن المعالج لا يستخدم `@Res()` أو `@Next()` من غير `passthrough`.

```ts
@Get('count')
async count(): Promise<number> {
  return this.members.count();
}
```

Nest تنتظر الـ Promise ثم تطبق قاعدة النوع على القيمة المحلولة.

يمكن إعادة `Observable`. Nest تشترك داخليًا وتأخذ آخر قيمة عند الاكتمال. عمق RxJS و `next.handle()` في فصل Interceptors. هنا يكفي أن التدفق المكتمل يعامل كنتيجة المعالج.

`@Header('Cache-Control', 'no-store')` يضبط رأسًا ثابتًا من غير الخروج إلى وضع المكتبة.

معالج `async` يعيد Promise دائمًا. هذا مدعوم مباشرة.

## Mental Model

```text
return { id: 'm-2' }     -> JSON   و 201 إن كان POST
return ['جديد']          -> JSON
return 4                 -> الجسم "4" من غير كائن JSON
return 'ready'           -> الجسم ready
return promise           -> انتظر ثم القاعدة نفسها
```

## أبسط مثال

```ts
import { Controller, Get, Header, Post } from '@nestjs/common';
import { MembersService } from './members.service.js';

@Controller('members')
export class MembersController {
  constructor(private readonly members: MembersService) {}

  @Get('count')
  @Header('Cache-Control', 'no-store')
  async count(): Promise<number> {
    return this.members.count();
  }

  @Post()
  create() {
    return { id: 'm-2', name: 'هدى' };
  }
}
```

## شرح المثال

- `count` عدد. Nest ترسل القيمة الأولية بعد انتظار الـ Promise، مع الرأس الثابت، ورمز 200.
- `@Get('count')` مسار ثابت. لا معامل `:id` هنا حتى لا يحجب كلمة `count`.
- `create` يعيد كائنًا على POST، فيخرج JSON برمز 201.
- الخدمة تملك العد والإنشاء. Controller يختار شكل القيمة المعادة.

## ماذا يحدث داخليًا؟

بعد انتهاء الدالة، طبقة الاستجابة تفحص القيمة. المصفوفة والكائن يدخلان `json`. القيم الأولية تُرسل مباشرة. `@HttpCode` وافتراض POST يحددان الرمز قبل الكتابة. `@Header` يضيف الرؤوس المعلنة على المعالج.

Interceptors على مسار العودة ترى هذه القيمة لأن الأسلوب القياسي ما زال شغالًا. تفاصيل ترتيبها في فصل Request Lifecycle. `@Res({ passthrough: true })` يبقي هذا المسار أيضًا، بينما `@Res()` وحده يوقفه.

إن رمت الخدمة استثناءً غير ممسوك، لا تُسلسل قيمة نجاح. Exception Filter يكتب استجابة الخطأ.

## مثال عملي

```ts
import { Controller, Get } from '@nestjs/common';
import { of, type Observable } from 'rxjs';
import { MembersService } from './members.service.js';

@Controller('members')
export class MemberLabelsController {
  constructor(private readonly members: MembersService) {}

  @Get('labels')
  labels(): Observable<string[]> {
    return of(this.members.labels());
  }

  @Get('banner')
  banner() {
    return 'المكتبة مفتوحة';
  }
}
```

`of` من RxJS يصدر المصفوفة ثم يكتمل. Nest تأخذ آخر إصدار وترسله JSON لأن المصفوفة كائن بالمعنى الذي يسلسله الإطار. `banner` نص يُكتب كما هو في الجسم. استيراد RxJS هنا ضيق؛ بناء تدفقات أوسع مؤجل إلى فصل Interceptors.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| توقع JSON لكائن عند `return 'ready'` | النص يُرسل خامًا |
| `@Res()` ثم `return { id }` من غير passthrough | الأسلوب القياسي متوقف على ذلك المسار |
| افتراض 200 لـ `POST` الذي يعيد كائنًا | الافتراضي 201 |
| إمساك كل الأخطاء بـ `try/catch` فارغ ثم `return null` كنجاح | العميل يرى نجاحًا. الفشل الذي تريد تحويله يُرمى ليصل إلى Exception Filter |

## متى نستخدمه؟

في مسارات REST التي تعيد JSON أو نصًا أو عددًا أو Promise لهذه الأشكال. هذا هو المسار الذي يوثّقه Nest كخيار موصى به.

## متى لا نستخدمه؟

عندما يجب أن تكتب تدفق CSV أو تنهي الرد بأدوات المنصة فقط. ذلك `@Res()` من غير passthrough، وأنت ترسل الجسم. تحويل العنوان يبقى `@Redirect` وهو ما زال أسلوب Nest لا `res.redirect`.

## مقارنة

| القيمة المعادة | الإرسال القياسي |
| --- | --- |
| كائن أو مصفوفة | JSON |
| `string` / `number` / `boolean` | القيمة كما هي |
| `Promise` | انتظار ثم القاعدة على الناتج |
| `Observable` مكتمل | آخر قيمة ثم القاعدة نفسها |

## تمارين

### تمرين سهل

ماذا يرسل `return { id: 'm-2' }` من معالج `@Post` من غير `@HttpCode`؟

<details><summary>الحل</summary>

JSON للكائن مع رمز 201.

</details>

### تمرين متوسط

كيف تمنع تخزين `GET /members/count` من غير `@Res()`؟

<details><summary>الحل</summary>

`@Header('Cache-Control', 'no-store')` على المعالج، وأعد العدد. الأسلوب القياسي يبقى.

</details>

### تمرين متقدم

`labels` تعيد `Observable` يصدر `['جديد']` ثم يكتمل. ماذا يصل إلى العميل، ولماذا؟

<details><summary>الحل</summary>

Nest تشترك وتأخذ آخر قيمة، وهي المصفوفة، فترسلها JSON. التدفق الذي لا يكتمل لا ينهي هذه الخطوة. بناء التدفقات الأعقد في فصل Interceptors.

</details>

## أسئلة مراجعة

- متى تُسلسل القيمة JSON؟
- ماذا يحدث لـ `async` يعيد `Promise<number>`؟
- أي Decorator رأس يبقى متوافقًا مع `return`؟

## الخلاصة

إعادة القيمة تترك Nest ترسل JSON للكائنات والمصفوفات، وترسل القيم الأولية كما هي، وتنتظر Promise وآخر قيمة في Observable. الرمز 200 أو 201 ما لم يغيّره `@HttpCode`.

## ماذا بعد؟

أمثلة تجمع مسارات هذا الفصل على موارد مختلفة.

## المصادر الرسمية

- NestJS — Controllers (Standard approach): https://docs.nestjs.com/controllers#routing
- NestJS — Asynchronicity: https://docs.nestjs.com/controllers#asynchronicity
- NestJS — Response headers: https://docs.nestjs.com/controllers#response-headers
