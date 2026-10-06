# التمهيد للفصل التالي

خطافات هذا الفصل تخص عمر العملية: إقلاع من الأوراق، ثم باب مفتوح، ثم إيقاف من الجذر إن استدعيت `app.close()` أو وصلت إشارة بعد `enableShutdownHooks()`.

حدّ واحد احمله معك: هذه الدوال لا تُستدعى على صنف `Scope.REQUEST`. عمر ذلك الصنف طلب واحد، لا عمر التطبيق. إن وضعت Prisma هناك لن ترى `$connect` في `onModuleInit`، وستدفع ثمن بناء الشجرة في كل طلب.

الفصل التالي يسمي النطاقات الثلاثة: `DEFAULT` وهو Singleton الذي عشت معه حتى الآن، و `REQUEST` الذي يعدي من يحقنه، و `TRANSIENT` الذي يعطي نسخة لكل مكان حقن. سترى لماذا تمرير المستخدم الحالي لا يحتاج نطاق طلب، ولماذا النسخة الواحدة من عميل القاعدة هي التي تستحق خطاف الإقلاع.

اكتب جملة قبل أن تفتح المجلد: أفتح المورد المشترك في `onModuleInit` على Singleton، وأغلقه في `onModuleDestroy`، ولا أعلّم الخدمة `REQUEST` كي أقرأ `user`.

## ما تثبته كل محطة

خريطة الدورة ترسم الاتجاه: الإقلاع من الأوراق إلى الجذر، والإيقاف عكسه. `onModuleInit` ينتظر اعتماديات الوحدة بعد حلها، فـ `$connect` يعيش هنا على `PrismaService` الورقة. `onApplicationBootstrap` يأتي بعد كل الوحدات وقبل `listen`، وهو مكان `ClientProxy.connect()` حتى لا يُحجز المنفذ قبل الناقل. `onModuleDestroy` بلا وسيط إشارة، وفيه `$disconnect` و `clearInterval`. `beforeApplicationShutdown(signal)` قبل إغلاق الاتصالات، علم إفراغ أو جاهزية. `onApplicationShutdown(signal)` آخر خطوة في `app.close()`. `enableShutdownHooks()` مطفأ افتراضيًا لأن المستمعين يكلفون ذاكرة. على Windows يصل `SIGINT` ولا يصل `SIGTERM`. `app.close()` لا يستدعي `process.exit` إن بقي مؤقت.

## سيناريو تعبر به

المتجر يغلق الحاوية بـ `SIGTERM`. السلسلة الصحيحة: الخطافات مفعّلة، Prisma ورقة، والفواتير مؤقت في الجذر يُمسح في `onModuleDestroy` قبل أن تصل الورقة إلى `$disconnect`. إن عكست الإغلاق تقطع القاعدة بينما الجذر ما زال يكتب.

```ts
await app.listen(process.env.PORT ?? 3000);
```

يأتي بعد `onApplicationBootstrap`. لا تضع `$connect` بعده وتتوقع أن الفشل يمنع فتح المنفذ: الفشل المتأخر يفتح بابًا على قاعدة ميتة.

أصناف `Scope.REQUEST` لا ترى أي خطاف من هذه القائمة. عمرها الطلب. نقل Prisma إلى ذلك النطاق يخطئ مكانين: الخطاف لا يعمل، والشجرة تُبنى كل مرة. الفصل التالي يسمي هذا الصعود عدوى.

## ماذا تقرأ هناك

`DEFAULT` هو ما استخدمته: نسخة واحدة، والخطافات تعمل، و Node ليس خيطًا لكل طلب. `REQUEST` نسخة لكل طلب وتعدي من يحقنها، بما في ذلك المتحكم. `TRANSIENT` نسخة لكل مكان حقن ولا تعدي، و `INQUIRER` يسمي الأب. درس الأداء يقيس إعادة بناء الشجرة. المزود الدائم (`durable: true`) يعيد الشجرة لمفتاح مثل `x-tenant-id` ولا يصلح لآلاف المفاتيح، ولا تمرر المستخدم الحالي عبره.

## تمرين عبور

خدمة كتالوج على الافتراضي، وحارس يملأ `request.user`. أين تقرأ المعرف، وأين تبقى `$connect`؟

<details>
<summary>الحل</summary>

المعرف من ديكور يقرأ `ExecutionContext` في المتحكم ويُمرر وسيطًا. `$connect` في `onModuleInit` على `PrismaService` بلا `scope`. لا تحقن `REQUEST` في الكتالوج.

</details>

## ملف تعبر به كما هو

الإقلاع يفعّل الخطافات ويستمع. Prisma ورقة تتصل وتقطع. الديكور يقرأ المستخدم من غير أن يعلّم الخدمة بنطاق طلب، وهذا أول ملف في الفصل التالي.

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks();
  await app.listen(process.env.PORT ?? 3000, '0.0.0.0');
}

void bootstrap();
```

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

```ts
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';

export const CurrentUserId = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest<{ user?: { id: string } }>();
    return request.user?.id ?? '';
  },
);
```

`BooksService.list(userId: string)` تبقى بلا `scope`. إن حقنت `REQUEST` داخلها صعد المتحكم، وسقط `onModuleInit` عن أي صنف في تلك السلسلة إن علّمته طلبًا أيضًا. Prisma ليست في تلك السلسلة.

تحقق قبل المغادرة: فشل `$connect` يمنع `listen`. `SIGTERM` يطبع الإغلاق على Linux. Ctrl+C على Windows يصل كـ `SIGINT`. مؤقت بلا `clearInterval` يبقي العملية بعد `app.close()`. لا خطاف على مزود طلب.

## قائمة قبل فتح النطاقات

1. ابحث عن `enableShutdownHooks` في `main.ts`.
2. ابحث عن `$connect` وتأكد أن الصنف بلا `scope`.
3. ابحث عن `setInterval` وتأكد من `clearInterval` في `onModuleDestroy`.
4. لا تضع `process.exit` داخل خطاف.
5. `ClientProxy.connect` في `onApplicationBootstrap` لا في `onModuleInit` إن كان يجب أن تكتمل الوحدات أولًا.
6. علم الإفراغ في `beforeApplicationShutdown` إن كان المنسّق يقرأ الجاهزية أثناء الإغلاق.
7. السجل الأخير في `onApplicationShutdown` بعد إغلاق المقابس.
8. اختبار يستدعي `app.close()` ويرى `$disconnect` حتى لو حذفت التفعيل مؤقتًا، ثم أعد التفعيل قبل التشغيل.
9. لا تنقل Prisma إلى طلب كي «يملك كل مستخدم اتصالًا».
10. اكتب جملة المستخدم: المعرف وسيط من الديكور، والعميل Singleton.
11. ترتيب الإقلاع: أوراق ثم جذر، وكل خطاف يُنتظر.
12. ترتيب الإيقاف: جذر ثم أوراق.
13. `onModuleDestroy` بلا إشارة، والخطافان الآخران يستقبلانها.
14. الحياة لا تُستدعى على `Scope.REQUEST`.

[Injection scopes](../40-injection-scopes/README.md)
