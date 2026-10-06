# Async Factories

## الهدف من هذا الدرس

جعل `useFactory` ترجع `Promise` حتى تنتظر الحاوية القيمة قبل بناء من يحقن الرمز.

## المشكلة التي يحلها

فتح عميل أو قراءة سر من مدير خارجي عملية غير متزامنة. إن رجعت المصنع كائنًا قبل `await`، فالخدمة تستعمل عميلًا غير متصل. إن فتحت في أول طلب، فأول زائر يرى الفشل الذي كان يجب أن يوقف الإقلاع. المصنع غير المتزامن جزء من بناء الرسم، وقد مر شكله في Asynchronous Providers. هنا مكانه بين أساليب المزود المخصص.

## الفكرة ببساطة

الدالة `async` أو التي ترجع `Promise` تُنتظر. الناتج المستقر يُخزن تحت `provide`. المستهلك يكتب المعامل كقيمة لا كوعد. رفض الوعد يوقف الإقلاع. الإغلاق ما زال مسؤوليتك على صنف له `onModuleDestroy` إن كانت القيمة كائن مكتبة.

## التعريف التقني

```ts
import { Inject, Injectable, Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

export const SEARCH = Symbol('SEARCH');

@Module({
  providers: [
    {
      provide: SEARCH,
      useFactory: async (config: ConfigService) => {
        const url = config.getOrThrow<string>('SEARCH_URL');
        await ping(url);
        return { url };
      },
      inject: [ConfigService],
    },
  ],
  exports: [SEARCH],
})
export class SearchModule {}

async function ping(url: string): Promise<void> {
  void url;
}

@Injectable()
export class SearchService {
  constructor(@Inject(SEARCH) private readonly endpoint: { url: string }) {}
}
```

`inject` يُحل قبل النداء، كما في المصنع المتزامن. `ConfigModule` مستورد أو عام حتى يوجد `ConfigService`.

هذا ليس `forRootAsync`. هناك المصنع يعيد خيارات ثم تولَّد مزودات الوحدة. هنا الرمز الواحد هو الناتج.

لا تجمع `useValue` مع الانتظار على نفس العنصر. ابنِ القيمة في المصنع. الاختبار الذي لا يريد شبكة يغطي `SEARCH` بـ `useValue` بقيمة جاهزة، فيتخطى المصنع.

`scope: Scope.REQUEST` على مصنع `async` يعيد الاتصال لكل طلب. أبقِ الافتراضي للعميل المشترك.

## Mental Model

```text
اعتماديات inject جاهزة
  -> await factory
  -> القيمة تحت Symbol
  -> بناء SearchService
رفض -> لا listen
```

## أبسط مثال

`SearchModule` أعلاه. `getOrThrow` أو فشل `ping` يمنع فتح المنفذ. `SearchService` تقرأ `endpoint.url` من غير `await` في الـ `constructor`. أضف خطاف الإغلاق على خدمة تغلف العميل إن استبدلت `{ url }` بمقبض شبكة.

## شرح المثال

`Symbol` من الدرس السابق يمنع تصادم `'SEARCH'`. المصنع يلتقط العنوان من الإعداد لا من حرف داخل الخدمة، فيقدر الاختبار على تغطية الرمز. `void url` في `ping` التعليمية مكان نداء حقيقي مثل `fetch` لعنوان الصحة. لا تبتلع الرفض بـ `try/catch` فارغ وإلا اكتمل الإقلاع بلا بحث.

## ماذا يحدث داخليًا؟

1. الحاوية ترى أن المصنع يعيد وعدًا.
2. تبني `ConfigService` ثم تستدعي الدالة.
3. تعلق بناء من يعتمد على `SEARCH` حتى التسوية.
4. تخزن `{ url }` كـ Singleton.
5. تبني `SearchService`.
6. كائن `{ url }` بلا خطاف. عملية الإيقاف لا تغلق مقبضًا لم يُخزن داخل صنف `OnModuleDestroy`.

## مثال عملي

غلاف يملك الإغلاق:

```ts
import { Injectable, OnModuleDestroy } from '@nestjs/common';

@Injectable()
export class SearchClient implements OnModuleDestroy {
  constructor(readonly url: string) {}

  async onModuleDestroy(): Promise<void> {
    await this.close();
  }

  private async close(): Promise<void> {
    return undefined;
  }
}
```

المصنع: `const client = new SearchClient(url); await client.open(); return client;` والرمز `provide: SearchClient`. مع `enableShutdownHooks` يصل `SIGTERM` إلى `onModuleDestroy`. هذا يجمع المصنع غير المتزامن مع فصل دورة الحياة. `useClass` وحده لا ينتظر `open` قبل أن يحقن الجيران إلا إذا كان الانتظار داخل `onModuleInit` على الصنف نفسه. اختر خطاف الصنف عندما يكون الزوج فتحًا وإغلاقًا على النوع، ومصنعًا غير متزامن عندما تريد القيمة جاهزة قبل بناء المعتمدين.

## أخطاء شائعة

- `await` ناقص داخل المصنع فيُخزن اتصال معلّق إن رجعت الكائن خطأً قبل الانتظار.
- تغطية الاختبار تنسى الرمز فتفتح شبكة.
- نطاق طلب على المصنع.
- التقاط الخطأ وإرجاع `null` ثم تعجب المستهلك.
- اعتبار هذا بديلًا عن `forRootAsync` لوحدة فيها خمسة مزودات. كرر المصنع لكل رمز أو ابنِ الوحدة الديناميكية.

## متى نستخدمه؟

رمز واحد قيمته تحتاج `await` مرة عند الإقلاع.

## متى لا نستخدمه؟

تهيئة صنف مسجّل أصلًا: `onModuleInit` أوضح للإغلاق. خيارات وحدة كاملة: `forRootAsync` أو `registerAsync`.

## مقارنة

| المكان | ماذا يُنتظر | من يُغلق |
| --- | --- | --- |
| `useFactory` غير متزامن | قيمة هذا الرمز | غلاف بخطاف إن لزم |
| `onModuleInit` | عمل على نسخة بُنيت | `onModuleDestroy` على الصنف نفسه |
| `forRootAsync` | كائن خيارات | مزودات الوحدة |

## تمارين

### تمرين سهل

هل معامل `SearchService` من نوع `Promise`؟

<details>
<summary>الحل</summary>

لا. الإطار ينتظر المصنع ويحقن القيمة المستقرة.

</details>

### تمرين متوسط

كيف يتخطى الاختبار `ping` الشبكي؟

<details>
<summary>الحل</summary>

يغطي الرمز `SEARCH` بـ `{ provide: SEARCH, useValue: { url: 'http://search.test' } }`.

</details>

### تمرين متقدم

متى تفضّل `onModuleInit` على مصنع يرجع `SearchClient`؟

<details>
<summary>الحل</summary>

عندما يكون الصنف مزودًا طبيعيًا وتريد `open` و `close` متجاورين على النوع نفسه بعد حقنه. المصنع أنسب لقيمة مكتبة أو عندما يجب ألا يُبنى المعتمد قبل انتهاء `await`.

</details>

## أسئلة مراجعة

- ماذا يوقف الإقلاع؟
- لماذا الرمز `Symbol` هنا متسق مع الدرس السابق؟
- ماذا لا يغلقه كائن عادي عند `SIGTERM`؟

## الخلاصة

المصنع غير المتزامن يؤخر من يعتمد على الرمز حتى تستقر القيمة. غطّه في الاختبار، وأغلق المقبض بصنف له خطاف إيقاف.

## ماذا بعد؟

أمثلة تجمع الأساليب الأربعة والرموز.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
