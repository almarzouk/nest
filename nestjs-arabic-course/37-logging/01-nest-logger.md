# Nest Logger

## الهدف من هذا الدرس

استبدال `console.log` بـ `Logger` من Nest حتى يظهر السياق والمستوى في مخرجات التطوير.

## المشكلة التي يحلها

سطر `published 42` بلا اسم الخدمة يضيع بين سطور الإقلاع. `console.error` لغياب كتاب يبدو كسقوط العملية. المستويات تفرّق «حدث متوقع» عن «يجب أن يوقظك». `Logger` يضيف اسم الصنف ويميز `log` و `warn` و `error`.

## الفكرة ببساطة

أنشئ `new Logger(BooksService.name)` في الخدمة. استخدم `log` للحدث الطبيعي، و `warn` لما اكتمل الطلب مع نقص غير قاتل، و `error` للفشل. لا تطبع الرمز ولا جسم الناشر. التصفية حسب المستوى تتم عند إنشاء التطبيق لا بحذف الأسطر.

## التعريف التقني

```ts
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class BooksService {
  private readonly logger = new Logger(BooksService.name);

  async publish(bookId: string) {
    this.logger.log(`publishing ${bookId}`);
    await this.prisma.book.update({
      where: { id: bookId },
      data: { status: 'PUBLISHED' },
    });
    this.logger.log(`published ${bookId}`);
  }
}
```

`BooksService.name` يبقى صحيحًا بعد التصغير إن حُفظ اسم الصنف، وهو أوضح من سلسلة `'BooksService'` المكررة. المستويات التي ستراها في المخرجات تشمل أيضًا `debug` و `verbose` و `fatal` على `Logger` الحديث. ابدأ بثلاثة: `log` و `warn` و `error`.

`error` يقبل رسالة ثم أثر المكدس:

```ts
this.logger.error(`isbn failed ${isbn}`, error instanceof Error ? error.stack : undefined);
```

لا تمرر كائن Axios كاملًا كرسالة. فيه ترويسات.

في `main.ts` يمكن خفض الضوضاء:

```ts
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn', 'log'],
});
```

أضف `'debug'` محليًا عند تتبع القص، ولا تتركه في إنتاج مزدحم بلا سبب.

## Mental Model

```text
Logger(اسم الصنف)
  log    مسار سعيد
  warn   عالق أو ناقص
  error  فشل مع مكدس
```

## أبسط مثال

```ts
if (counts.waiting > 100) {
  this.logger.warn(`covers waiting ${counts.waiting}`);
}
```

هذا مكان `console.log` في نبضة الطابور. الرقم سياق كافٍ. لا تطبع قائمة معرفات الكتب كلها كل 30 ثانية.

## شرح المثال

`warn` يظهر حتى لو أخفيت `debug`. الشرط يمنع سطرًا كل نبضة عندما يكون الطابور هادئًا. اسم `QueueWatchService` يظهر من `new Logger(QueueWatchService.name)` حتى تعرف أن السطر من النبضة لا من `POST`.

## ماذا يحدث داخليًا؟

`Logger` الافتراضي يكتب إلى الطرفية بتنسيق Nest: مستوى، سياق، رسالة. هو لا يفتح ملفًا ولا يرسل إلى مجمع. التجميع لاحقًا عبر مسجّل مخصص يكتب JSON إلى المخرج القياسي وتلتقطه المنصة. في الاختبار، السطور الافتراضية تزعج المخرج. يمكن استبدال المسجّل في اختبار التطبيق أو تجاهل السجل حتى فصل الاختبار.

`fatal` لحالة ستوقف العملية. فشل ISBN لكتاب واحد `error` على الطلب لا `fatal` على التطبيق.

## مثال عملي

مستويات المتجر:

| الحدث | المستوى |
| --- | --- |
| نُشر `bookId` | `log` |
| طابور أغلفة فوق الحد | `warn` |
| فشل `add` بعد الحفظ | `error` مع المكدس |
| Redis المخبأ سقط ثم أكملنا من الناشر | `warn` |
| تعذر الإقلاع بلا `ISBN_TOKEN` | الفشل من `getOrThrow` قبل الطلبات |

لا مستوى لـ «دخلت الدالة» في الإنتاج. ذلك `debug` مؤقت.

## أخطاء شائعة

- `console.log` بجانب `Logger` في الخدمة نفسها.
- طباعة `ISBN_TOKEN` أو `authorization`.
- `error` لكل كتاب غير موجود. الغياب `warn` أو لا سطر إن كان `404` متوقعًا بكثرة.
- `fatal` لفشل طلب واحد.

## متى نستخدمه؟

في الخدمات والعاملين والمستمعين حيث وقع أثر: نشر، إدراج مهمة، فشل قص.

## متى لا نستخدمه؟

بدل رمي الاستثناء. السجل يشرح، و `NotFoundException` يجاوب الزائر. السطر بلا رمي يخفي الفشل عن المرشح.

## مقارنة

| الأداة | ماذا تضيف |
| --- | --- |
| `console.log` | نص بلا مستوى ثابت |
| `Logger` | مستوى وسياق الصنف |
| رمي الاستثناء | رد HTTP أو فشل مهمة |

## تمارين

### تمرين سهل

كيف تسمي سياق `Logger` لخدمة الكتب؟

<details>
<summary>الحل</summary>
`new Logger(BooksService.name)`.
</details>

### تمرين متوسط

أي مستوى لطابور ينتظر أكثر من الحد وما زال يقص؟

<details>
<summary>الحل</summary>
`warn`. العمل مستمر والوضع غير سليم.
</details>

### تمرين متقدم

لماذا لا تمرر `AxiosError` كاملًا إلى `logger.error`؟

<details>
<summary>الحل</summary>
لأن الكائن قد يحتوي ترويسة المفتاح وجسم الناشر. سجّل الرمز و `isbn` والمكدس إن لزم.
</details>

## أسئلة مراجعة

- ما الفرق بين `log` و `error`؟
- أين تُضبط مستويات الإقلاع؟
- ماذا يبقى مسؤولية الاستثناء لا السجل؟

## الخلاصة

`Logger` يضع مستوىً واسم صنف على الحدث. لا تطبع أسرارًا، ولا تستبدل السجل بالرد الذي ينتظره الزائر.

## ماذا بعد؟

مسجّل بصنفك أنت عندما لا يكفي تنسيق الطرفية الافتراضي.

## المصادر الرسمية

- NestJS — Logger: https://docs.nestjs.com/techniques/logger
