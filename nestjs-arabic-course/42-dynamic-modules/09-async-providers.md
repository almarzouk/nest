# Asynchronous Providers

## الهدف من هذا الدرس

تسجيل مزود بمصنع `useFactory` يرجع `Promise`، حتى ينتظر Nest اكتمال الاتصال قبل حقن الناتج في الخدمات.

## المشكلة التي يحلها

`forRootAsync` يبني خيارات وحدة. أحيانًا الوحدة ثابتة والمزود نفسه هو الذي يتصل: عميل يُفتح بـ `await client.connect()` ثم يُحقن كقيمة. إن رجعت المصنع كائنًا قبل انتهاء `connect`، فأول مستخدم يرى عميلًا غير جاهز. المصنع غير المتزامن يجعل الوعد جزءًا من الإقلاع.

## الفكرة ببساطة

في `providers` ضع كائنًا: `provide` هو الرمز، و `useFactory` دالة `async`، و `inject` اختياري. الإطار يحل الحقن، يستدعي الدالة، وينتظرها، ثم يخزن القيمة الناتجة تحت الرمز. من يحقن الرمز يأخذ القيمة لا الوعد.

## التعريف التقني

```ts
import { Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Module({
  providers: [
    {
      provide: 'ASYNC_CONNECTION',
      useFactory: async (config: ConfigService) => {
        const url = config.getOrThrow<string>('DATABASE_URL');
        return { url, ready: true };
      },
      inject: [ConfigService],
    },
  ],
  exports: ['ASYNC_CONNECTION'],
})
export class ConnectionModule {}
```

```ts
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class BooksService {
  constructor(
    @Inject('ASYNC_CONNECTION')
    private readonly connection: { url: string; ready: boolean },
  ) {}
}
```

هذا مزود مخصص من نوع المصنع، ودرس `useFactory` في فصل Custom Providers يفصّل الأنواع الأربعة. هنا التركيز على الانتظار: رجوع `Promise` يؤخر من يعتمد على الرمز حتى التسوية.

إن رمى المصنع أو رُفض الوعد، يفشل الإقلاع كما يفشل `onModuleInit` الذي يرمي. ضع الاتصال الإلزامي هنا أو في الخطاف، لا في أول طلب حيث يظهر الفشل متأخرًا.

الفرق عن `registerAsync`: هناك المصنع يعيد خيارات وحدة ثم تُبنى مزودات الوحدة. هنا المصنع يعيد قيمة المزود نفسها.

يمكن الجمع: `forRootAsync` يمرر العنوان، ومصنع غير متزامن داخل الوحدة يفتح العميل. لا تكرر `await connect` في الاثنين.

أغلق المورد في `onModuleDestroy` على خدمة تملكه. قيمة `useValue` المنتَجة لا تملك خطافًا إن كانت كائنًا عاديًا. لذلك غالبًا المصنع يعيد صنفًا فيه `onModuleDestroy`، أو خدمة مجاورة تغلق ما خُزّن.

## Mental Model

```text
inject -> await useFactory -> القيمة في الحاوية -> بناء من يحقن الرمز
رفض الوعد -> الإقلاع يتوقف
```

المستهلك لا يكتب `await` على الحقن. الانتظار تم قبل البناء.

## أبسط مثال

المقطعان أعلاه. `BooksService` ترى `ready: true` لأن البناء لم يبدأ لها قبل تسوية المصنع. صدّر الرمز `'ASYNC_CONNECTION'` إن كانت الخدمة في وحدة أخرى، كما في أي مزود مخصص.

## شرح المثال

`inject: [ConfigService]` يتطلب أن تكون الخدمة مرئية: استيراد `ConfigModule` في هذه الوحدة أو جعلها عامة. ترتيب المعاملات يطابق المصفوفة. الكائن الراجع هو ما يُحقن، لا `ConfigService`.

## ماذا يحدث داخليًا؟

1. الحاوية ترى مصنعًا يرجع وعدًا.
2. تبني اعتماديات `inject` أولًا.
3. تنتظر `Promise`.
4. تسجل القيمة كـ Singleton تحت الرمز ما لم تغيّر `scope`.
5. تبني `BooksService` وتمرر القيمة إلى `@Inject`.
6. الإغلاق لا يعرف القيمة إلا إن ربطتها بخدمة لها خطاف.

## مثال عملي

عميل Redis يُفتح قبل الخدمات:

```ts
import { Module } from '@nestjs/common';

class RedisClient {
  async connect(url: string): Promise<void> {
    void url;
  }
}

@Module({
  providers: [
    {
      provide: RedisClient,
      useFactory: async () => {
        const client = new RedisClient();
        await client.connect(process.env.REDIS_URL ?? 'redis://localhost:6379');
        return client;
      },
    },
  ],
  exports: [RedisClient],
})
export class RedisModule {}
```

في Docker يكون العنوان اسم الخدمة `redis` لا `localhost`، وذلك فصل لاحق. هنا المصنع ينتظر `connect` قبل أن يحقن أي مستهلك العميل. أغلق في خدمة تغلف العميل وتنفّذ `onModuleDestroy`، أو انقل `connect` إلى `onModuleInit` على `@Injectable()` عادي إن كان ذلك أوضح من مصنع. اختر خطافًا عندما تريد زوج الفتح والإغلاق على الصنف نفسه، ومصنعًا غير متزامن عندما القيمة ليست صنف Nest بل كائن مكتبة.

## أخطاء شائعة

- نسيان `await` داخل المصنع فيُحقن وعدًا مستقرًا خطأً إن رجعت الوعد من غير أن تكون الدالة مسار الانتظار الذي يتوقعه الإطار. اجعل الدالة `async` أو أرجع `Promise` التي تنتظر الاتصال فعلًا.
- حقن الرمز من غير `exports` في وحدة أخرى.
- فتح الاتصال في المصنع وإغلاقه في كل طلب.
- خلط هذا النمط مع تعليم المزود `REQUEST` فيتكرر الاتصال لكل طلب.
- قراءة `localhost` من داخل صورة Compose.

## متى نستخدمه؟

قيمة تُنتَج بعملية غير متزامنة مرة عند الإقلاع ثم تُشارك.

## متى لا نستخدمه؟

صنف `@Injectable()` يكفي مع `onModuleInit`. خيارات وحدة كاملة مكانها `forRootAsync` لا مصنع معزول يخفي بقية المزودات.

## مقارنة

| الأسلوب | ماذا يُنتظر | ماذا يُحقن |
| --- | --- | --- |
| `useFactory` غير متزامن | ناتج المصنع | القيمة تحت `provide` |
| `forRootAsync` | كائن خيارات الوحدة | مزودات الوحدة بعد ذلك |
| `onModuleInit` | داخل الصنف بعد بنائه | الصنف نفسه بُني قبل الانتظار |

## تمارين

### تمرين سهل

هل تكتب `await` عند حقن `'ASYNC_CONNECTION'` في الخدمة؟

<details>
<summary>الحل</summary>

لا. الإطار انتظر المصنع قبل بناء الخدمة. المعامل هو القيمة.

</details>

### تمرين متوسط

ما فرق ناتج المصنع هنا عن ناتج مصنع `registerAsync`؟

<details>
<summary>الحل</summary>

هنا الناتج هو قيمة المزود. في `registerAsync` الناتج خيارات تُستخدم لتسجيل مزودات الوحدة.

</details>

### تمرين متقدم

لماذا كائن المكتبة الراجع من المصنع لا يغلق نفسه عند `SIGTERM`؟

<details>
<summary>الحل</summary>

خطافات العمر على أصناف Nest المسجّلة كمزودات تنفّذ الواجهة. الكائن العادي بلا `onModuleDestroy` يبقى مفتوحًا. غلفه بخدمة تغلقه، وفعّل `enableShutdownHooks`.

</details>

## أسئلة مراجعة

- أين يوضع `inject`؟
- ماذا يحدث لو رُفض الوعد؟
- متى تفضّل `onModuleInit` على المصنع؟

## الخلاصة

المصنع غير المتزامن يؤخر الإقلاع حتى تستقر قيمة المزود. حقن الرمز يعطي القيمة. أغلق المورد في خدمة لها خطاف إن لم يكن الكائن نفسه مزود خطافات.

## ماذا بعد؟

أمثلة تجمع `forRoot` و `register` وإعادة التصدير.

## المصادر الرسمية

- NestJS — Dynamic modules: https://docs.nestjs.com/fundamentals/dynamic-modules
- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
