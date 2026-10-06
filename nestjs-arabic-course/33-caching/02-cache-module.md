# CacheModule

## الهدف من هذا الدرس

تسجيل المخبأ كوحدة حتى يُحقن في خدمة ISBN، من الحزمة الحالية لا من `@nestjs/common`.

## المشكلة التي يحلها

`Map` على الخدمة تعمل حتى تتعدد النسخ والمسارات. Nest توفر `CacheModule` حتى يكون المخزن مزوّدًا قابلًا للاستبدال باختبار أو بـ Redis. استيراده من المكان القديم يثبّت API نُقلت.

## الفكرة ببساطة

الحزمة `@nestjs/cache-manager` تصدّر `CacheModule`. `register` يضبط المخزن في الذاكرة افتراضيًا. الوحدة التي تستورده تحقن الرمز `CACHE_MANAGER`. اجعله متاحًا حيث تُحسب النتيجة، لا في كل متحكم.

## التعريف التقني

```bash
npm i @nestjs/cache-manager cache-manager
```

```ts
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { IsbnService } from './isbn.service.js';

@Module({
  imports: [
    CacheModule.register({
      ttl: 10 * 60 * 1000,
    }),
  ],
  providers: [IsbnService],
})
export class IsbnModule {}
```

`ttl` في التوثيق الحالي بالميلي ثانية. `10 * 60 * 1000` عشر دقائق. القيمة `0` أو غياب `ttl` يعني عدم انتهاء الصلاحية في السلوك الافتراضي لـ cache-manager، فلا تتركه بلا قصد.

`isGlobal: true` إن كان المخزن واحدًا لكل التطبيق. من غيره استورد `CacheModule` في كل وحدة تحقن المدير، أو صدّر الوحدة. لتطبيق بدأ بمخبأ ISBN فقط، التسجيل داخل `IsbnModule` يكفي.

`CacheModule` الذي كان في `@nestjs/common` نُقل. الاستيراد الحالي من `@nestjs/cache-manager`.

## Mental Model

```text
IsbnModule
  CacheModule.register ttl بالميلي ثانية
  IsbnService تحقن CACHE_MANAGER
```

## أبسط مثال

تسجيل عالمي في الجذر عندما تشترك أكثر من وحدة في السياسة نفسها:

```ts
CacheModule.register({
  isGlobal: true,
  ttl: 10 * 60 * 1000,
});
```

وحدة البريد التي لا تخزّن لا تتأثر بالمزوّد الزائد كثيرًا، لكن المهلة العالمية قد لا تناسب كل مفتاح. يمكن تمرير مدة عند `set` لاحقًا وتبقى هذه القيمة افتراضًا.

## شرح المثال

`register` بلا `stores` يبقي التخزين في ذاكرة العملية عبر cache-manager. لا Redis في هذا الدرس. إعادة تشغيل `nest start` تفرغ المفاتيح، وهذا مفيد في التطوير حتى لا تطارح عنوانًا قديمًا.

## ماذا يحدث داخليًا؟

الوحدة الديناميكية تنشئ مزوّد `CACHE_MANAGER`. الحقن بالرمز لا بالاسم النصي. كل `get` و `set` يمر على المخزن الافتراضي. مع `isGlobal` النسخة واحدة في التطبيق. نسختان من العملية ما زالتا ذاكرتين. الإعداد لا يفتح منفذًا خارجيًا.

## مثال عملي

أبقِ `HttpModule.registerAsync` بجانب `CacheModule.register` في `IsbnModule`. الأول للناشر والثاني للنسخة. لا تضع `CacheModule` داخل المتحكم ولا تعتمد على `CacheInterceptor` العالمي بعد. المعترض التلقائي على `GET` قد يحفظ جسمًا خاصًا بمستخدم إن نُسي شكل المفتاح. التخزين الصريح في `lookup` أوضح لعقد ISBN.

## أخطاء شائعة

- `import { CacheModule } from '@nestjs/common'`.
- `ttl: 600` باعتقاد أنها ثوانٍ، فتنتهي القيمة خلال 600 ميلي ثانية.
- `isGlobal` ثم مفاتيح بلا بادئة فتصطدم وحدة البريد مع ISBN.
- افتراض أن التسجيل وحده يخزّن نداء `lookup`. لا بد من `get` و `set`.

## متى نستخدمه؟

عندما قررت مفتاحًا ومدة لنتيجة قراءة، وتريد استبدال المخزن لاحقًا بلا تغيير توقيع `lookup`.

## متى لا نستخدمه؟

قبل وجود نتيجة تتكرر. تسجيل عالمي «لأن التطبيقات الناضجة تخزّن» يترك مفاتيح بلا مالك.

## مقارنة

| الاستيراد | الحالة |
| --- | --- |
| `@nestjs/cache-manager` | التسجيل الحالي |
| `@nestjs/common` | المكان القديم المنقول |
| `Map` داخل الخدمة | تجربة محلية بلا استبدال |

## تمارين

### تمرين سهل

من أي حزمة يُستورد `CacheModule`؟

<details>
<summary>الحل</summary>

من `@nestjs/cache-manager`.

</details>

### تمرين متوسط

كم تساوي `ttl: 600` في الإعداد الحالي؟

<details>
<summary>الحل</summary>

600 ميلي ثانية لا 600 ثانية.

</details>

### تمرين متقدم

لماذا لا يكفي `register` ليخفف نداءات ISBN؟

<details>
<summary>الحل</summary>

لأن التسجيل يوفر المخزن. `IsbnService` ما زالت تستدعي الناشر حتى تقرأ المفتاح وتكتبه بعد النجاح.

</details>

## أسئلة مراجعة

- ما وحدة TTL في `register`؟
- ماذا يفعل `ttl` الافتراضي إن تُرك بلا انتهاء؟
- لماذا أُجّل `CacheInterceptor`؟

## الخلاصة

`CacheModule.register` من `@nestjs/cache-manager` يجهّز مخزنًا بمدة بالميلي ثانية. الحفظ الفعلي في الخدمة التي تعرف المفتاح.

## ماذا بعد؟

`get` و `set` عبر `CACHE_MANAGER` حول `lookup`.

## المصادر الرسمية

- NestJS — Caching: https://docs.nestjs.com/data/caching
