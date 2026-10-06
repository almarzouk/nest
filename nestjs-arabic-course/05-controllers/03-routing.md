# Routing

## الهدف من هذا الدرس

كيف يتكوّن المسار الكامل، وبأي ترتيب تُختار الدالة عندما يطابق أكثر من نمط الطلب نفسه.

## المشكلة التي يحلها

`GET /authors/me` قد يبتلعه `@Get(':id')` المكتوب قبله. التطبيق يقلع بلا تحذير، والخطأ يظهر عند الطلب. Routing يشرح الدمج وترتيب التسجيل.

## الفكرة ببساطة

المسار = بادئة Controller + نص Decorator الطريقة. على Express، وهو المحول الافتراضي، ترتيب الإعلان يحدد من يفوز. NestJS 12 يضيف سياسة تعارض واختيار ترتيب حسب التحديد، وكلاهما اختياري.

## التعريف التقني

نقطة النهاية هي طريقة HTTP زائد المسار. `@Controller('authors')` مع `@Get('me')` يعني `GET /authors/me`. `@Get()` وحده يعني `GET /authors`. ما بعد `?` لا يدخل المطابقة.

الافتراضي `routeResolutionStrategy: 'declaration'`: التسجيل بترتيب الإعلان. مسار بمعامل يسبق مسارًا ثابتًا فيحجبه على Express. `ParseIntPipe` لا ينقذ الموقف لأن اختيار المعالج يتم قبل أي Pipe.

NestJS 12 يضيف خيارين ضمن `NestApplicationOptions`. الافتراضي يبقي السلوك السابق.

```ts
routeConflictPolicy: { duplicate: 'off', shadow: 'off' }
routeResolutionStrategy: 'declaration'
```

القيم `'warn'` و `'error'` لكل من `duplicate` و `shadow`. مع `'error'` تُجمع الأزواج في `RouteConflictException` واحدة عند `app.init()` أو عند `app.listen()` إن لم تُستدعَ `init`.

| المفتاح | المعنى |
| --- | --- |
| `duplicate` | الطريقة والمسار والمضيف والإصدار متطابقة |
| `shadow` | نمطان يمكن أن يطابقا الطلب نفسه، مثل `/authors/me` و `/authors/:id` |

`'specificity'` يسجل الأكثر تحديدًا أولًا: قطعة ثابتة، ثم معامل، ثم wildcard. الأنواع `RouteConflictPolicy` و `RouteResolutionStrategy` مصدَّرة من `@nestjs/common`.

Express حساس للترتيب. Fastify (`find-my-way`) يرتب بالتحديد أصلًا، فـ `shadow` و `'specificity'` بلا أثر عنده، بينما `duplicate` يعمل على المحولين.

## Mental Model

```text
إعلان:  @Get(':id') ثم @Get('me')
Express الافتراضي: "me" يطابق :id ولا يصل إلى me

routeResolutionStrategy: 'specificity'
  الثابت me يُسجَّل قبل :id
```

## أبسط مثال

```ts
import { Controller, Get } from '@nestjs/common';

@Controller('authors')
export class AuthorsController {
  @Get('me')
  me() {
    return { id: 'self' };
  }

  @Get(':id')
  findOne() {
    return { id: 'from-param' };
  }
}
```

## شرح المثال

- `me` ثابت و `:id` معامل. بترتيب الإعلان هذا، `GET /authors/me` يصل إلى `me` على Express.
- لو انعكس الإعلان، `:id` يلتقط `me` أولًا مع الاستراتيجية الافتراضية.
- اسم الدالة ما زال بلا دور في المطابقة.
- الشرطة `-` والنقطة `.` في المسار النصي حرفان عاديان.

## ماذا يحدث داخليًا؟

عند الإقلاع تُسجَّل المسارات. مع `declaration` يبقى ترتيب الملف. مع `specificity` تعاد ترتيبها قبل التسليم للمحول الحساس للترتيب. سياسة `shadow: 'warn'` تطبع التعارض وتكمل. `shadow: 'error'` توقف الإقلاع بعد جمع كل الأزواج.

Wildcard: `@Get('media/*')` يطابق `media/` و `media/a/b`. طبقة التوافق في Nest تسمح بالنجمة غير المسماة رغم أن Express 5 الخام يريد اسمًا مثل `media/{*splat}`. النجمة في وسط المسار على Express تحتاج اسمًا: `ab{*splat}cd`. Fastify لا يدعم النجمة في الوسط.

`app.setGlobalPrefix('api')` يسبق بادئة Controller، فيصبح `GET /api/authors/me`.

## مثال عملي

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

const app = await NestFactory.create(AppModule, {
  routeConflictPolicy: { duplicate: 'error', shadow: 'warn' },
  routeResolutionStrategy: 'specificity',
});

app.setGlobalPrefix('api');
await app.listen(3000);
```

بهذا الإعداد، تكرار `GET /api/authors/me` مرتين يوقف الإقلاع، وحجب `me` خلف `:id` يظهر تحذيرًا، والتسجيل يقدّم القطعة الثابتة.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| `@Get(':id')` قبل `@Get('me')` على Express الافتراضي | `GET /authors/me` يدخل معالج المعامل |
| الاعتماد على Pipe لرفض كلمة `me` | الـ Pipe يعمل بعد اختيار المعالج |
| `*` في وسط المسار على Fastify | النمط غير مدعوم هناك |
| توقع أن `?page=2` يغيّر اختيار الدالة | الاستعلام يُقرأ بعد المطابقة |

## متى نستخدمه؟

عند تركيب أي مسار أطول من البادئة، وعند وجود ثابت ومعامل تحت البادئة نفسها. فعّل `shadow: 'warn'` في التطبيقات التي كبر فيها عدد المسارات.

## متى لا نستخدمه؟

`routeResolutionStrategy` لا يغيّر سلوك Fastify في الحجب. على Fastify اجعل `duplicate` هو الفحص الذي تعتمد عليه، ورتب المعنى في تصميم المسار نفسه.

## مقارنة

| الاستراتيجية | من يفوز بين `me` و `:id` |
| --- | --- |
| `declaration` | من كُتب أولًا في التسجيل |
| `specificity` | القطعة الثابتة `me` ثم المعامل |

## تمارين

### تمرين سهل

ما المسار الكامل لـ `@Controller('authors')` و `@Get('me')` مع `setGlobalPrefix('api')`؟

<details><summary>الحل</summary>

`GET /api/authors/me`.

</details>

### تمرين متوسط

لماذا يقلع التطبيق بينما `GET /authors/me` يعيد مؤلفًا رقمه `me`؟

<details><summary>الحل</summary>

`@Get(':id')` مُسجَّل قبل `@Get('me')` والاستراتيجية `declaration` على Express. المطابقة الأولى تفوز. Pipe لا يعيد الاختيار.

</details>

### تمرين متقدم

ما الذي يميّز `duplicate: 'error'` عن `shadow: 'warn'` على Fastify؟

<details><summary>الحل</summary>

`duplicate` يعمل على Fastify وExpress ويوقف الإقلاع عند تطابق الطريقة والمسار والمضيف والإصدار. `shadow` بلا أثر على Fastify لأن موجهه يرتب بالتحديد. التحذير مفيد على Express حيث الترتيب يقرر.

</details>

## أسئلة مراجعة

- ممّ يتكوّن مسار المعالج؟
- ما الافتراضي لـ `routeResolutionStrategy`؟
- أين تعمل النجمة في وسط المسار؟

## الخلاصة

المسار بادئة زائد قطعة الطريقة. على Express الافتراضي الترتيب في الملف هو الحكم، وNestJS 12 يجعل التعارض ظاهرًا عند الإقلاع بسياسة تختارها.

## ماذا بعد؟

Decorators الطرق: `@Get` و `@Post` و `@Put` و `@Patch` و `@Delete`.

## المصادر الرسمية

- NestJS — Controllers (Routing): https://docs.nestjs.com/controllers#routing
- NestJS — Route conflicts: https://docs.nestjs.com/controllers#route-conflicts-and-resolution-order
