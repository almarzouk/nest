# Custom Logger

## الهدف من هذا الدرس

تمرير مسجّل من تنفيذك إلى Nest عبر `LoggerService` حتى تستخدمه الإطار وخدماتك بالشكل نفسه.

## المشكلة التي يحلها

`new Logger()` في الخدمات يكتب بتنسيق Nest، بينما أخطاء الإقلاع من الإطار تكتب بتنسيق آخر إن بدّلت المكتبة في مكان واحد فقط. المنصة التي تجمع السجلات تريد مخرجًا واحدًا. `app.useLogger` يوجّه الإطار إلى تنفيذك، و `Logger.overrideLogger` يوجّه استدعاءات `Logger` الثابتة أثناء الإقلاع.

## الفكرة ببساطة

صنف يطبق `LoggerService`: `log` و `error` و `warn` و `debug` و `verbose`، و `fatal` إن كنت على إصدار يعرّفه. في `main.ts` أنشئ التطبيق بخيار `bufferLogs: true` ثم `useLogger` حتى لا تضيع أسطر ما قبل ربط المسجّل. التنفيذ في هذا الدرس يكتب سطرًا نصيًا واحدًا. JSON في الدرس التالي.

## التعريف التقني

```ts
import { LoggerService } from '@nestjs/common';

export class AppLogger implements LoggerService {
  log(message: unknown, context?: string) {
    this.write('log', message, context);
  }

  error(message: unknown, stack?: string, context?: string) {
    this.write('error', message, context, stack);
  }

  warn(message: unknown, context?: string) {
    this.write('warn', message, context);
  }

  debug(message: unknown, context?: string) {
    this.write('debug', message, context);
  }

  verbose(message: unknown, context?: string) {
    this.write('verbose', message, context);
  }

  fatal(message: unknown, context?: string) {
    this.write('fatal', message, context);
  }

  private write(level: string, message: unknown, context?: string, stack?: string) {
    const text = typeof message === 'string' ? message : JSON.stringify(message);
    process.stdout.write(`${level} ${context ?? '-'} ${text}${stack ? `\n${stack}` : ''}\n`);
  }
}
```

```ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(new AppLogger());
await app.listen(3000);
```

`bufferLogs` يحتفظ برسائل الإقلاع المبكرة ثم يفرغها في المسجّل بعد `useLogger`. من غيره تضيع رسائل أو تخرج بالتنسيق الافتراضي قبل الربط.

الخدمات تبقى على `new Logger(BooksService.name)`. بعد الربط، تلك الاستدعاءات تصل إلى تنفيذك إن كان التعيين يشملها عبر `useLogger` كما في سلوك Nest الحالي. لا تنشئ `console.log` موازيًا داخل `write`.

## Mental Model

```text
bufferLogs
  useLogger(AppLogger)
    الإطار والخدمات -> write
      stdout
```

## أبسط مثال

تجاهل `debug` خارج التطوير داخل `write`:

```ts
if (level === 'debug' && process.env.NODE_ENV === 'production') return;
```

الشرط في مكان واحد. الخدمات تستمر في نداء `debug` بلا `if` حول كل سطر.

## شرح المثال

`error` في الواجهة يضع المكدس في الوسيط الثاني والسياق في الثالث، وهذا ترتيب `LoggerService` لا ترتيب `console.error`. تبديلهما يخفي اسم الصنف ويطبع المكدس كسياق. التزم التوقيع.

## ماذا يحدث داخليًا؟

`NestFactory` يخزن الرسائل عندما يكون `bufferLogs` مفعلًا. `useLogger` يستبدل المسجّل ويدفع المخزن. بعدها `Logger` الداخلي يستدعي طرائق صنفك. إن رمى `write`، قد يخفي الخطأ الأصلي الذي أردت تسجيله. اجعل الكتابة لا ترمي. فشل المجمع لاحقًا يجب ألا يسقط `POST`.

لا تطلب Prisma من المسجّل عند كل سطر. المسجّل تحت الطلب، واستعلام داخله يضاعف الحمل ويولد سجلات عن السجل.

## مثال عملي

اختبار يتأكد أن التحذير خرج بلا تشغيل الخادم كاملًا:

```ts
const lines: string[] = [];
const logger = new AppLogger();
logger.write = (level, message) => lines.push(`${level}:${String(message)}`);
```
إن بقي `write` خاصًا فاستخرج الدالة. لا تعتمد على قراءة الطرفية في الوحدة.

## أخطاء شائعة

- `useLogger` بعد `listen` فتخرج بداية الإقلاع بتنسيق مختلف.
- نسيان `fatal` أو `debug` فتسقط استدعاءات الإطار وقت التشغيل.
- رمي خطأ من `write`.
- تسجيل من داخل `write` عبر `Logger` فيدور النداء.

## متى نستخدمه؟

عندما تجمع المنصة stdout وتريد تحكمًا في التنسيق لكل أسطر Nest لا لخدماتك فقط.

## متى لا نستخدمه؟

لمكتبة ثقيلة قبل أن ترى سطرًا ناقصًا في التنسيق الافتراضي. ابدأ بـ `Logger` ثم بدّل التنفيذ عندما يفرض الإنتاج شكلًا.

## مقارنة

| الربط | ماذا يوحّد |
| --- | --- |
| `new Logger` فقط | خدماتك، وتنسيق Nest |
| `useLogger` + `bufferLogs` | الإطار وخدماتك بعد الإقلاع المبكر |
| `console` داخل الخدمة | قناة ثالثة تتجنبها |

## تمارين

### تمرين سهل

أي واجهة يطبقها `AppLogger`؟

<details>
<summary>الحل</summary>
`LoggerService` من `@nestjs/common`.
</details>

### تمرين متوسط

لماذا `bufferLogs: true` قبل `useLogger`؟

<details>
<summary>الحل</summary>
حتى تُحفظ رسائل الإقلاع الأولى وتُطبع بالمسجّل الجديد لا بالتنسيق الافتراضي أو تُفقد.
</details>

### تمرين متقدم

لماذا لا يستعلم المسجّل القاعدة عند كل `log`؟

<details>
<summary>الحل</summary>
لأن السجل على مسار الطلب. الاستعلام يبطئ كل نداء وقد يولّد أخطاء تسجيل بلا نهاية.
</details>

## أسئلة مراجعة

- ما ترتيب وسائط `error`؟
- ماذا لا يرمي `write`؟
- أي طرائق تبقيها حتى لا ينكسر الإطار؟

## الخلاصة

`LoggerService` تنفيذ واحد، و `useLogger` بعد `bufferLogs` يمرر إليه أسطر الإطار. الكتابة لا ترمي ولا تلمس القاعدة.

## ماذا بعد؟

السطر نفسه كـ JSON فيه حقول ثابتة تستطيع البحث عنها.

## المصادر الرسمية

- NestJS — Logger: https://docs.nestjs.com/techniques/logger
