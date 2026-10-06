# Factory Providers

## الهدف من هذا الدرس

بناء قيمة بـ `useFactory` عندما يعتمد الإنشاء على مزودات أخرى أو على خطوة غير `new` لصنف واحد.

## المشكلة التي يحلها

عميل البحث يحتاج `SearchOptions` المسجّلة، وربما اسم منطقة قد يغيب. `useValue` قيمة ثابتة قبل أن تُحل الخيارات. `useClass` يبني صنفًا ولا يعيد كائن مكتبة حسبما تقرره دالة. المصنع دالة ترجع القيمة بعد أن تمرر Nest وسائط `inject`.

## الفكرة ببساطة

`useFactory` هي الدالة، و `inject` مصفوفة رموز تُحل وتُمرَّر بالترتيب نفسه. ما ترجعه الدالة هو قيمة الرمز في `provide`. الدالة قد تكون `async`، وNest تنتظر الوعد لأن المصانع المتزامنة وغير المتزامنة مدعومة.

## التعريف التقني

```ts
{
  provide: 'SEARCH_CLIENT',
  useFactory: (options: SearchOptions, region?: string) => {
    return new SearchClient(options.host, region ?? 'jeddah');
  },
  inject: [SearchOptions, { token: 'SEARCH_REGION', optional: true }],
}
```

العنصر العادي في `inject` إلزامي. الكائن `{ token, optional: true }` يجعل الغياب `undefined` لذلك الوسيط. القائمتان متوازيتان: أول رمز أول معامل.

المصنع الذي لا يحتاج أحدًا يترك `inject` فارغة أو يحذفها.

تصدير الرمز النصي يكون `exports: ['SEARCH_CLIENT']` أو تصدير كائن الوصفة نفسه. الرمز هو هوية التصدير.

`SearchOptions` تبقى مسجّلة في `providers` بجانب الوصفة، باختصار أو بغيره. المصنع لا يسجلها نيابة عنك.

## Mental Model

```text
حل SearchOptions
حل SEARCH_REGION أو undefined
    -> useFactory(options, region)
    -> SearchClient
    -> يُخزَّن تحت SEARCH_CLIENT
```

## أبسط مثال

```ts
import { Injectable, Module } from '@nestjs/common';
import { SearchService } from './search.service.js';

@Injectable()
export class SearchOptions {
  readonly host = 'https://search.internal';
}

const searchClientProvider = {
  provide: 'SEARCH_CLIENT',
  useFactory: (options: SearchOptions) => {
    return { host: options.host, limit: 20 };
  },
  inject: [SearchOptions],
};

@Module({
  providers: [SearchOptions, searchClientProvider, SearchService],
})
export class SearchModule {}
```

## شرح المثال

- القيمة المرجَعة كائن بسيط فيه المضيف والحد. الرمز نص، لذلك ستحقنه الخدمة بـ `@Inject('SEARCH_CLIENT')`.
- `inject: [SearchOptions]` يسبق نداء الدالة. داخل الدالة لا تستدعِ الحاوية يدويًا.
- ترتيب `providers` لا يغني عن `inject`.
- الحد `20` قرار المصنع، لا حقلًا يقرأه Controller من الاستعلام.

## ماذا يحدث داخليًا؟

عند حل `'SEARCH_CLIENT'` تحل Nest كل عنصر في `inject` أولًا، بما في ذلك المصانع التي تعتمد عليها تلك الرموز، ثم تستدعي الدالة. إن أعادت `Promise` تنتظرها قبل تخزين القيمة تحت الرمز. مع Singleton يتم ذلك مرة عند الإقلاع. مع `scope: Scope.REQUEST` على الوصفة يتكرر النداء لكل طلب، والعدوى التي عرفتها في Scope تبقى سارية.

وسيط اختياري غائب يصل `undefined`. وسيط إلزامي غير مسجّل يفشل الإقلاع برسالة حل الاعتماديات.

## مثال عملي

```ts
import { Injectable, Module } from '@nestjs/common';

@Injectable()
export class SearchOptions {
  get host() {
    return process.env.SEARCH_HOST ?? 'https://search.internal';
  }
}

export const searchClientProvider = {
  provide: 'SEARCH_CLIENT',
  useFactory: async (options: SearchOptions, region?: string) => {
    const client = new SearchClient(options.host);
    await client.warmup(region ?? 'jeddah');
    return client;
  },
  inject: [SearchOptions, { token: 'SEARCH_REGION', optional: true }],
};

class SearchClient {
  constructor(private readonly host: string) {}

  async warmup(region: string) {
    return { host: this.host, region };
  }
}

@Module({
  providers: [SearchOptions, searchClientProvider],
  exports: ['SEARCH_CLIENT'],
})
export class SearchModule {}
```

`warmup` تعمل مرة ضمن Singleton قبل إكمال الإقلاع. `'SEARCH_REGION'` إن سُجّلت بـ `useValue` تُمرَّر كنص؛ إن غابت تبقى جدة. ضع `SearchClient` في ملفها عند نموها. وجودها هنا يوضح أن `new` داخل المصنع ملكك، بينما `SearchOptions` ملك الحاوية.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| ترتيب `inject` مختلف عن ترتيب المعاملات | المنطقة تصل مكان `SearchOptions` |
| نسيان تسجيل `SearchOptions` | فشل الحل قبل نداء المصنع |
| `useFactory` تقرأ Provider بـ `new` بدل `inject` | نسخة ثانية خارج السجل |
| تصدير `SearchClient` بينما `provide` نص | المستورد يبحث عن رمز لم يُسجَّل |

## متى نستخدمه؟

عندما تتكون القيمة من مزودات، أو من خطوة غير متزامنة، أو من كائن مكتبة وسائطه محسوبة. الاتصال الذي يحتاج `warmup` مثال واضح.

## متى لا نستخدمه؟

لصنف تبنيه Nest كما هو. `useClass` أقصر. ولثابت لا يقرأ مزودًا. `useValue` يكفي. المصنع الذي يعيد `new MailNotifier()` من غير وسائط إضافية هو `useClass` في صورة أطول.

## مقارنة

| | `useFactory` | `useClass` |
| --- | --- | --- |
| الناتج | ما ترجعه الدالة | نسخة الصنف |
| الوسائط | مصفوفة `inject` بالترتيب | `constructor` للصنف المبني |
| عمل غير متزامن | الدالة `async` تُنتظَر | البناء نفسه متزامن والتهيئة اللاحقة في حياة الصنف |

## تمارين

### تمرين سهل

من أين تأتي معاملات `useFactory`؟

<details><summary>الحل</summary>

من رموز `inject` بعد حلها، بالترتيب نفسه.

</details>

### تمرين متوسط

كيف تسمح بغياب `'SEARCH_REGION'` من غير إسقاط الإقلاع؟

<details><summary>الحل</summary>

ضعها في `inject` بهذا الشكل: `{ token: 'SEARCH_REGION', optional: true }`، وتعامل الوسيط المقابل كقيمة قد تكون `undefined`.

</details>

### تمرين متقدم

لماذا `await client.warmup()` داخل مصنع Singleton يؤخر `listen`، وما الذي يتغير مع `scope: Scope.REQUEST`؟

<details><summary>الحل</summary>

الحل يتم عند الإقلاع للـ Singleton، فالإقلاع ينتظر الوعد. مع `REQUEST` يتكرر المصنع لكل طلب، فزمن `warmup` يدخل في زمن ذلك الطلب، والسلسلة التي تحقن الرمز تصبح لكل طلب.

</details>

## أسئلة مراجعة

- ماذا يخزن الرمز: الدالة أم قيمة إرجاعها؟
- كيف يُعلَم عنصر `inject` الاختياري؟
- كيف تصدّر رمزًا نصيًا أنتجه المصنع؟

## الخلاصة

`useFactory` دالة ترجع قيمة الرمز بعد حل `inject` بالترتيب. الوعد يُنتظَر، والوسيط الاختياري كائن `{ token, optional: true }`، وباقي الرموز تبقى مسجّلة بجانب الوصفة.

## ماذا بعد؟

اسم ثانٍ للنسخة نفسها: Existing Providers.

## المصادر الرسمية

- NestJS — Factory providers: https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory
