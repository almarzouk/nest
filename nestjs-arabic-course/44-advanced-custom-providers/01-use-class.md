# useClass

## الهدف من هذا الدرس

ربط رمز حقن بصنف تختار أنت أي تنفيذ يُبنى له، وهو الشكل الصريح لاختصار `providers: [BooksService]`.

## المشكلة التي يحلها

الاختصار يخفي أن الرمز والبناء شيئان. عندما تريد `ConfigService` في التطوير و `ConfigService` آخر في الإنتاج، أو بديلًا يطبق العقد نفسه، تحتاج `provide` ثابتًا و `useClass` متغيرًا. Nest يبني الصنف المختار ويحقنه لمن طلب الرمز، لا لمن طلب اسم الملف.

## الفكرة ببساطة

```ts
providers: [
  { provide: ConfigService, useClass: ProductionConfigService },
]
```

من يكتب `constructor(private readonly config: ConfigService)` يحصل على `ProductionConfigService`. الصنفان يتشاركان شكل الدوال التي تستدعيها. إن كان الرمز صنفًا مجردًا يبقى وقت التشغيل، فالتنفيذ `implements` ذلك الصنف.

## التعريف التقني

الاختصار `providers: [CatsService]` يساوي `{ provide: CatsService, useClass: CatsService }`. التوثيق يسمي هذا مزودًا قياسيًا.

`useClass` يبني نسخة جديدة من الصنف المذكور كلما احتاج الرمز نسخة حسب `scope`. الافتراضي Singleton: نسخة واحدة من صنف الإنتاج خلف رمز `ConfigService`. هذا يختلف عن `useExisting` الذي لا يبني ثانيًا بل يشير إلى رمز مبني أصلًا.

اختيار البيئة عند تحميل الملف:

```ts
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service.js';
import { DevelopmentConfigService } from './development-config.service.js';
import { ProductionConfigService } from './production-config.service.js';

const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
  exports: [ConfigService],
})
export class ConfigHostModule {}
```

صدّر الرمز `ConfigService` لا اسم الصنف التنفيذي فقط، لأن المستهلك يطلب الرمز. يمكن تصدير كائن المزود كاملًا كما في التوثيق.

الصنف التنفيذي `@Injectable()` إن كان سيُبنى بالحاوية وتُحقن اعتمادياته. `useClass` ما زال يحل `constructor` ذلك الصنف.

## Mental Model

```text
الرمز الذي يطلبه المستهلك  ->  provide
الصنف الذي يُبنى          ->  useClass
النسخة                    ->  واحدة في DEFAULT
```

تغيير `useClass` لا يغيّر ملفات المستهلك.

## أبسط مثال

المقطع أعلاه. `BooksService` تحقن `ConfigService` وتنادي `get(key)`. في `NODE_ENV=development` التنفيذ يقرأ قيمًا محلية. في غيره يقرأ مصدر الإنتاج. الاختبار يمرر `{ provide: ConfigService, useClass: DevelopmentConfigService }` داخل `Test.createTestingModule` حتى لو كانت البيئة مختلفة.

## شرح المثال

المتغير `configServiceProvider` تنظيم فقط. الشرط يعمل مرة عند تقييم الوحدة، لا لكل طلب. إن أردت التبديل لكل طلب فأنت في نطاق طلب، وهو قرار أثقل من اختيار بيئة الإقلاع. صدّر الرمز حتى ترى الوحدات الأخرى التنفيذ.

## ماذا يحدث داخليًا؟

1. الحاوية تسجل الرمز `ConfigService` مرتبطًا بالصنف المختار.
2. عند أول حقن تبني ذلك الصنف وتحل اعتمادياته.
3. تخزن النسخة للرمز.
4. أي `useClass` لاحق للرمز نفسه في وحدة تغطي التسجيل يعيد التوجيه. اعرف أي وحدة تملك الربط.
5. الاختبار الذي يعيد `{ provide: ConfigService, useValue: fake }` يغلب الصنف. ذلك درس `useValue`.
6. الواجهة الممحية لا توضع في `provide`. استعمل صنفًا مجردًا أو `Symbol`.

## مثال عملي

تخزين ملفات: `FileStore` مجرد، و `DiskFileStore` أو `S3FileStore` في `useClass` حسب الإعداد عند الإقلاع. المتحكم يحقن `FileStore`. لا فرع `if (s3)` داخل المتحكم. أغلق عميل S3 في `onModuleDestroy` على التنفيذ الذي فتحه.

## أخطاء شائعة

- تصدير الصنف التنفيذي ونسيان الرمز الذي يحقنه الآخرون.
- شرط `NODE_ENV` داخل كل دالة بدل مرة في المزود.
- `useClass` لصنف غير `@Injectable` وفيه اعتماديات لن تُحل.
- توقع نسخة جديدة لكل مستهلك. ذلك `scope: Scope.TRANSIENT` على كائن المزود.
- وضع واجهة TypeScript في `provide`.

## متى نستخدمه؟

تبديل تنفيذ لعقد واحد عند الإقلاع أو في الاختبار، مع بناء الحاوية للصنف.

## متى لا نستخدمه؟

كائن جاهز أو مزيف بلا `constructor`. ذلك `useValue`. اسم ثانٍ لنسخة موجودة. ذلك `useExisting`.

## مقارنة

| الحقل | من يبني | متى |
| --- | --- | --- |
| `useClass` | الحاوية | تنفيذ قابل للحقن |
| `useValue` | أنت قبل التسجيل | ثابت أو مزيف |
| `useExisting` | لا أحد من جديد | لقب لرمز موجود |
| `useFactory` | دالتك | تجميع من عدة مزودات |

## تمارين

### تمرين سهل

ما معادل `providers: [BooksService]` الطويل؟

<details>
<summary>الحل</summary>

`{ provide: BooksService, useClass: BooksService }`.

</details>

### تمرين متوسط

لماذا يُصدَّر `ConfigService` لا `ProductionConfigService` فقط؟

<details>
<summary>الحل</summary>

لأن المستهلكين يحقنون الرمز. تصدير التنفيذ وحده لا يخدم من كتب نوع الأساس في الـ `constructor`.

</details>

### تمرين متقدم

اختبار يريد قرصًا وهميًا بينما الوحدة ربطت `S3FileStore`. ماذا تضع في `TestingModule`؟

<details>
<summary>الحل</summary>

`{ provide: FileStore, useClass: FakeFileStore }` أو `useValue` لكائن الدوال. الربط في الاختبار يغطي رمز `FileStore`.

</details>

## أسئلة مراجعة

- ماذا يطلب المستهلك: الرمز أم ملف التنفيذ؟
- متى يُقيَّم شرط `NODE_ENV` في المثال؟
- ما الفرق المختصر عن `useExisting`؟

## الخلاصة

`useClass` يسمي الصنف الذي يُبنى خلف الرمز. الاختصار الشائع حالته الخاصة عندما الرمز والصنف واحد. بدّل التنفيذ من التسجيل لا من داخل المتحكم.

## ماذا بعد؟

`useValue`: قيمة جاهزة خلف الرمز، ومنها مزيف الاختبار.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
