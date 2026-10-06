# إعداد العميل

## الهدف من هذا الدرس

نقل عنوان الناشر والمهلة والمفتاح إلى إعداد الوحدة عبر `registerAsync`، وترك القيم الافتراضية آمنة.

## المشكلة التي يحلها

`baseURL` المثبّت في `IsbnModule` يكسر التشغيل عند تغيير البيئة، ويدفع المفتاح إلى Git. مهلة `800` المنسوخة في المعترض وفي Axios تنحرف عندما يعدّل مطور واحدة فقط. الإعداد مصدر واحد تقرأه المصنع عند الإقلاع.

## الفكرة ببساطة

`ConfigModule` يقرأ البيئة. `HttpModule.registerAsync` يحقن `ConfigService` ويبني خيارات Axios. الخدمة لا تنادي `process.env`. الاختبار يمرر إعدادًا قصير المهلة أو يستبدل `IsbnService` كلها.

## التعريف التقني

مفاتيح مقترحة:

```text
ISBN_BASE_URL=https://isbn.example/v1
ISBN_TIMEOUT_MS=800
ISBN_TOKEN=...
```

```ts
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { IsbnService } from './isbn.service.js';

@Module({
  imports: [
    HttpModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        baseURL: config.getOrThrow<string>('ISBN_BASE_URL'),
        timeout: Number(config.get<string>('ISBN_TIMEOUT_MS') ?? 800),
        maxRedirects: 2,
        headers: {
          authorization: `Bearer ${config.getOrThrow<string>('ISBN_TOKEN')}`,
          accept: 'application/json',
        },
      }),
    }),
  ],
  providers: [IsbnService],
  exports: [IsbnService],
})
export class IsbnModule {}
```

`getOrThrow` يوقف الإقلاع إن غاب العنوان أو المفتاح، وذلك أفضل من أول طلب زائر يفشل بـ `401` غامض. `Number(...)` لأن بيئة النظام نصوص. إن كان الناتج `NaN` أصلح المفتاح قبل التشغيل.

عميل `fetch` في `@nestjs/http-client` يستعمل `baseUrl` ومهلة قد تُكتب `'800ms'`. لا تخلط أسماء الحقول بين الحزمتين في الملف نفسه.

## Mental Model

```text
بيئة
  ConfigService
    useFactory
      Axios instance داخل HttpModule
        IsbnService.lookup بلا عناوين
```

## أبسط مثال

اختبار وحدة لا يفتح الشبكة: استبدل الخدمة.

```ts
const isbn = { lookup: async () => ({ title: 'NestJS in Practice' }) };
```

اختبار يمر على `HttpService` يستبدل الناقل أو يشغّل خادمًا محليًا على `ISBN_BASE_URL` لا على ناشر حقيقي. المفتاح في الاختبار قيمة وهمية.

## شرح المثال

المصنع يعمل مرة عند إنشاء الوحدة. تغيير ملف البيئة بعد الإقلاع لا يحدّث Axios حتى تعيد التشغيل، ما لم تبنِ مصنعًا يقرأ كل مرة، وهذا غير لازم هنا. `accept: application/json` يعلن ما تستطيع ترجمته. إن أعاد الناشر XML ستفشل فحص `title` بـ `502`، وهذا أصدق من تحليل صامت.

## ماذا يحدث داخليًا؟

Nest ينشئ سياق الوحدة، يحل `ConfigService`، وينتظر المصنع إن كان `async`. الناتج يُمرر كخيارات منشئ Axios. كل `get` نسبي يلصق `baseURL` ويمرر الترويسات. `timeout` يبدأ مع إرسال الطلب. إعادة التوجيه محدودة بـ `maxRedirects` حتى لا يتبع العميل سلسلة مفتوحة. لا تسجّل `headers.authorization` في معترض Axios للتصحيح على بيئة مشتركة.

## مثال عملي

قيم مختلفة بلا نسخ الوحدة:

```text
تطوير   ISBN_BASE_URL=http://localhost:4010  ISBN_TIMEOUT_MS=200
إنتاج   ISBN_BASE_URL=https://isbn.example/v1  ISBN_TIMEOUT_MS=800
```

الاختبار المحلي يستعمل مهلة قصيرة حتى يفشل التعليق بسرعة. الإنتاج يمنح الناشر 800ms كما قرر منتج صفحة الإنشاء. الكود واحد.

## أخطاء شائعة

- التزام `.env` وفيه `ISBN_TOKEN`.
- `baseUrl` على خيارات Axios. الحقل في Axios هو `baseURL`.
- قراءة `process.env` داخل `lookup` بعد أن وفرت المصنع.
- مهلة المعترض 800 ومهلة Axios 5000 فيصمت الزائر على حد المعترض بينما المقبس ما زال مفتوحًا، أو العكس.

## متى نستخدمه؟

كلما اختلف العنوان أو السر بين البيئات. أي عميل صادر في المتجر.

## متى لا نستخدمه؟

لمضيف ثابت داخل اختبار لا يقرأ بيئة المطور. هناك `register` بقيم صريحة في وحدة الاختبار أوضح من متغيرات خفية.

## مقارنة

| الأسلوب | متى |
| --- | --- |
| `register({...})` | اختبار أو عنوان لن يتغير في المستودع |
| `registerAsync` + `ConfigService` | تطوير وإنتاج |
| `HttpClientModule` | مسار `fetch` في التوثيق الحالي، بأسماء حقول مختلفة |

## تمارين

### تمرين سهل

أي دالة تمنع الإقلاع عند غياب `ISBN_BASE_URL`؟

<details>
<summary>الحل</summary>
`config.getOrThrow`.
</details>

### تمرين متوسط

لماذا `timeout` في البيئة نص ويُمرر رقمًا؟

<details>
<summary>الحل</summary>
متغيرات البيئة نصوص. `Number` ينتج الميلي ثانية التي يتوقعها Axios.
</details>

### تمرين متقدم

ما خطر مهلة RxJS أقصر من مهلة Axios على المسار نفسه؟

<details>
<summary>الحل</summary>
الزائر يرى الخطأ بينما طلب Axios ما زال يعمل حتى مهلة المقبس، فيُحمَّل الناشر بلا مستفيد.
</details>

## أسئلة مراجعة

- أي حقل Axios لعنوان الأساس؟
- لماذا لا تُصدَّر الترويسة إلى سجلات التصحيح؟
- من يقرأ الإعداد: `lookup` أم المصنع؟

## الخلاصة

`registerAsync` يبني عميل ISBN من البيئة عند الإقلاع. المهلة والمفتاح والعنوان لا تُنسخ داخل الدالة، والخدمة تبقى ترجمة واستثناءات.

## ماذا بعد؟

تخزين نتيجة تستحق الإعادة، ومتى تصبح النسخة المحفوظة خطرًا.

## المصادر الرسمية

- NestJS — HTTP module (Axios): https://docs.nestjs.com/techniques/http-module
- NestJS — Configuration: https://docs.nestjs.com/techniques/configuration
