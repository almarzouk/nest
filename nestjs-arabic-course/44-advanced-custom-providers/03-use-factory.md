# useFactory

## الهدف من هذا الدرس

بناء قيمة المزود بدالة تستقبل مزودات أخرى، عندما لا يكفي `useClass` ولا قيمة ثابتة.

## المشكلة التي يحلها

عميل قاعدة يحتاج خيارات من `ConfigService` ثم `new DatabaseConnection(options)`. الصنف وحده لا يرى الإعداد إلا إذا حقنته أنت. المصنع يصرح `inject` فيستلم النسخ ويعيد القيمة التي تُخزن تحت الرمز.

## الفكرة ببساطة

`useFactory` دالة، و `inject` مصفوفة رموز تُمرر معاملات بالترتيب نفسه. الناتج هو ما يُحقن لاحقًا. الدالة قد ترجع `Promise`، وذلك درس Async Factories. هنا الناتج متزامن حتى ترى الربط.

## التعريف التقني

شكل التوثيق، مع اعتماد اختياري:

```ts
import { Module } from '@nestjs/common';
import { DatabaseConnection } from './database.connection.js';
import { OptionsProvider } from './options.provider.js';

const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider, label?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options, label ?? 'default');
  },
  inject: [OptionsProvider, { token: 'LABEL', optional: true }],
};

@Module({
  providers: [connectionProvider, OptionsProvider],
  exports: ['CONNECTION'],
})
export class DatabaseModule {}
```

`{ token, optional: true }` يجعل الغياب `undefined` بدل فشل الإقلاع. الرموز الإلزامية في `inject` يجب أن تكون مسجّلة في الوحدة أو مستوردة.

المستهلك:

```ts
import { Inject, Injectable } from '@nestjs/common';
import { DatabaseConnection } from './database.connection.js';

@Injectable()
export class BooksRepository {
  constructor(@Inject('CONNECTION') private readonly db: DatabaseConnection) {}
}
```

صدّر `'CONNECTION'` أو كائن المزود كاملًا. التوثيق يسمح بالاثنين.

لا تضع `useClass` مع `useFactory` على العنصر نفسه. أسلوب واحد لكل رمز.

المصنع يعمل عند بناء Singleton مرة. إن علّمت المزود `Scope.REQUEST` فالمصنع يتكرر لكل طلب، ومعه كلفة الشجرة.

## Mental Model

```text
حل inject من اليسار إلى اليمين
  -> استدعاء useFactory
  -> تخزين الناتج تحت provide
  -> حقن الناتج لا الدالة
```

## أبسط مثال

مقطع `connectionProvider`. `OptionsProvider` صنف `@Injectable` يعيد `{ url: string }`. المصنع ينشئ `DatabaseConnection`. `BooksRepository` لا تعرف `OptionsProvider` إن لم تحقنها. هذا عزل: تجميع العميل في مكان واحد.

## شرح المثال

الترتيب في `inject` هو العقد. تبديل العنصرين يمرر النص مكان المزود ويكسر الدالة عند التشغيل لا عند المترجم بالضرورة. الاعتماد الاختياري آخرًا أسهل قراءة. `exports: ['CONNECTION']` يطابق `provide`.

## ماذا يحدث داخليًا؟

1. الحاوية تبني `OptionsProvider` أولًا لأنه في `inject`.
2. تبحث عن `'LABEL'`. مع `optional` تكمل بلا قيمة.
3. تستدعي الدالة بالنسخ.
4. تخزن كائن `DatabaseConnection` تحت `'CONNECTION'`.
5. تبني `BooksRepository` وتمرر الكائن.
6. رمي المصنع يوقف الإقلاع. مناسب لعنوان ناقص، وقاسٍ لتكامل اختياري.

## مثال عملي

اختيار محوّل بريد من الإعداد من غير شرط داخل كل إرسال:

```ts
{
  provide: 'MAILER',
  useFactory: (config: ConfigService) => {
    const host = config.getOrThrow<string>('SMTP_HOST');
    return { host };
  },
  inject: [ConfigService],
}
```

الوحدة تستورد `ConfigModule`. الاختبار يستبدل الرمز بـ `useValue` ولا يشغّل المصنع. إن أردت الإعداد على مستوى مكتبة لا مزود واحد، ابقَ على `forRootAsync` الذي يسجل عدة مزودات من كائن خيارات.

## أخطاء شائعة

- اختلاف ترتيب `inject` عن المعاملات.
- نسيان تسجيل `OptionsProvider` في `providers` أو `imports`.
- مصنع ثقيل على `Scope.REQUEST`.
- إرجاع صنف لا نسخة: `return DatabaseConnection` بدل `return new DatabaseConnection(...)` إلا إن كان المقصود قيمة أخرى.
- منطق طلب داخل المصنع. لا `REQUEST` هنا إلا بقصد النطاق.

## متى نستخدمه؟

تجميع كائن من مزودات عند الإقلاع، أو إخفاء `new` لمكتبة خارجية.

## متى لا نستخدمه؟

تنفيذ بسيط بلا تجميع: `useClass`. ثابت اختبار: `useValue`. لقب لنسخة قائمة: `useExisting`.

## مقارنة

| الأسلوب | المدخل | الناتج |
| --- | --- | --- |
| `useFactory` | رموز `inject` | ما ترجعه الدالة |
| `useClass` | constructor الصنف | نسخة الصنف |
| `forRootAsync` | مصنع خيارات وحدة | وحدة فيها مزودات عدة |

## تمارين

### تمرين سهل

ماذا يُحقن في `BooksRepository`: الدالة أم ناتجها؟

<details>
<summary>الحل</summary>

ناتج `useFactory` المخزون تحت `'CONNECTION'`.

</details>

### تمرين متوسط

ماذا يفعل `optional: true` على عنصر في `inject`؟

<details>
<summary>الحل</summary>

إن غاب الرمز تُمرر `undefined` ولا يفشل الإقلاع بسبب هذا الرمز.

</details>

### تمرين متقدم

لماذا استبدال `'CONNECTION'` بـ `useValue` في الاختبار يلغي المصنع؟

<details>
<summary>الحل</summary>

لأن الرمز يُغطى بقيمة جاهزة في حاوية الاختبار، فلا حاجة لحل `inject` ولا لاستدعاء الدالة.

</details>

## أسئلة مراجعة

- ما علاقة ترتيب `inject` بالمعاملات؟
- كيف تُصدّر الرمز النصي؟
- متى المصنع أغلى من اللازم؟

## الخلاصة

`useFactory` يبني قيمة الرمز من مزودات تصرح بها. الناتج هو ما يُحقن. أبقِ المصنع على إقلاع Singleton ما لم تقس نطاقًا أضيق.

## ماذا بعد؟

`useExisting`: الرمز الثاني الذي يشير إلى النسخة نفسها.

## المصادر الرسمية

- NestJS — Custom providers: https://docs.nestjs.com/fundamentals/custom-providers
