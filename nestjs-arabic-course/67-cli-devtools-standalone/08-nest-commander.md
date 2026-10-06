# nest-commander

## الهدف من هذا الدرس

كتابة أمر `export-invoices` لرف بزخارف `nest-commander` فوق حاوية Nest، مع معرفة أن الحزمة من طرف ثالث.

## المشكلة التي يحلها

تحليل `process.argv` يدويًا يتفرع مع كل علم. أو الأمر ينشئ الخدمات بـ `new` فينقطع الحقن. أو نسيان أن `CommandFactory` يعطّل مسجّل Nest ما لم تطلبوه، فيختفي خطأ الإقلاع.

## الفكرة ببساطة

الصنف يمتد `CommandRunner` ويحمل `@Command`. `@Option` يحوّل علمًا إلى قيمة. الصنف `@Injectable` عبر معاملة الإطار له كمزود. `CommandFactory.run(AppModule)` من `nest-commander` يبني السياق ويشغّل الأمر ثم `close`. الوثائق تصف الحزمة كطرف ثالث غير مدار من فريق Nest الأساسي. الاختبار عبر `CommandTestFactory` في حزمة `nest-commander-testing`.

## التعريف التقني

`npm i nest-commander`. `run(passedParams: string[], options?: Record<string, unknown>): Promise<void>`. الاسم في كائن الخيارات يطابق `name` في `@Option`، والقيمة هي ما يعيده المحلّل. `CommandFactory.run(AppModule)` يعطّل المسجّل افتراضيًا. الوسيط الثاني مسجّل مخصص أو مستويات مثل `['warn', 'error']`. المصنع ينادي `NestFactory` ثم `app.close` عند الانتهاء. الأمر يُدرَج في `providers`. التشغيل من `main.ts` بدل `listen`. `--help` يأتي من commander. أخطاء الإقلاع تُلتقط حول `run` بـ `try/catch`.

## Mental Model

```text
nest-commander → CommandFactory.run
صنف أمر → run(المواضع، الخيارات)
المنظمة علم محلول لا جسم طلب
```

## أبسط مثال

```ts
import { Command, CommandFactory, CommandRunner, Option } from 'nest-commander';
import { AppModule } from './app.module.js';
import { InvoiceExportJob } from './billing/invoice-export.job.js';

@Command({ name: 'export-invoices', description: 'Export one library invoices' })
export class ExportInvoicesCommand extends CommandRunner {
  constructor(private readonly job: InvoiceExportJob) {
    super();
  }

  async run(_params: string[], options?: { organizationId?: string }): Promise<void> {
    if (!options?.organizationId) return;
    await this.job.exportOrganization(options.organizationId);
  }

  @Option({ flags: '-o, --organization-id <id>', description: 'Library id' })
  parseOrganizationId(value: string): string {
    return value;
  }
}

await CommandFactory.run(AppModule, ['error']);
```

سجّلوا `ExportInvoicesCommand` و `InvoiceExportJob` في الوحدة.

## شرح المثال

- `parseOrganizationId` يعيد النص الذي يصبح `options.organizationId`.
- المعرف وسيط تشغيل تتحققون منه مقابل سجل منصة، لا حقل يرسله متصفح.
- `['error']` يبقي أخطاء Nest ظاهرة. بلا الوسيط الثاني تصمت السجلات.

## ماذا يحدث داخليًا؟

1. commander يطابق الاسم والخيارات.
2. كل `@Option` يحوّل النص قبل `run`.
3. `CommandFactory` يبني سياق تطبيق مستقل فيحقن `InvoiceExportJob`.
4. بعد `run` يُغلق التطبيق حتى لا تبقى المقابض.
5. فشل بناء الوحدة يُرمى من `run`؛ التقطوه وحدّدوا رمز خروج غير صفر حتى لا تبدو المهمة ناجحة في Cron.

## مثال عملي

اختبار الأمر يستبدل المهمة:

```ts
import { CommandTestFactory } from 'nest-commander-testing';

const moduleRef = await CommandTestFactory.createTestingCommand({
  imports: [BillingModule],
})
  .overrideProvider(InvoiceExportJob)
  .useValue({ exportOrganization: async () => 'ok' })
  .compile();
```

الشكل يطابق `Test.createTestingModule`: `overrideProvider` ثم `compile`. لا يشغّل SMTP ولا Stripe. المعرف في الاختبار مكتبة وهمية.

## أخطاء شائعة

- نسيان تسجيل الصنف في `providers` فيُبنى التطبيق بلا أمر.
- اعتبار الحزمة جزءًا من `@nestjs/core`. مسائلها تُفتح على مستودع nest-commander.
- طباعة كل السجلات الحساسة بتمكين المسجّل كاملًا على آلة مشتركة.
- الأمر يقرأ `tenantId` من ملف يرفعه المستخدم بلا تحقق.

## متى نستخدمه؟

أوامر تشغيل يكررها الفريق: تصدير، إعادة محاولة صندوق، فحص تهجير، ولها خيارات وأسماء.

## متى لا نستخدمه؟

لاستدعاء واحد أثناء التطوير؛ REPL أخف. ولواجهة HTTP. إن كفى `createApplicationContext` بلا أعلام فلا تضيفوا حزمة طرف ثالث.

## مقارنة

| الأسلوب | التحليل | الإغلاق |
| --- | --- | --- |
| `process.argv` | يدوي | عليكم |
| سياق مستقل | بلا أوامر | `app.close()` |
| `CommandFactory.run` | `@Command` و `@Option` | المصنع يغلق |

## تمارين

### تمرين سهل

من يملك الحزمة حسب الوثائق؟

<details>
<summary>الحل</summary>

طرف ثالث. الصفحة تطلب الإبلاغ في مستودع nest-commander لا في قضايا Nest الأساسية.

</details>

### تمرين متوسط

لماذا السجلات صامتة في أول تشغيل؟

<details>
<summary>الحل</summary>

`CommandFactory` يعطّل المسجّل ما لم تمرروا مسجّلًا أو قائمة مستويات كوسيط ثانٍ.

</details>

### تمرين متقدم

كيف تختبرون الأمر بلا قاعدة؟

<details>
<summary>الحل</summary>

`CommandTestFactory.createTestingCommand` ثم `overrideProvider(InvoiceExportJob)` ثم `compile`. الحزمة تستخدم `@nestjs/testing` تحت الغطاء.

</details>

## أسئلة مراجعة

- أي صنف يُمتد وأي زخرفة على الصف؟
- ماذا يعيد `@Option`؟
- متى يُستدعى `close`؟
- أين تُسجَّل الأعطال؟

## الخلاصة

`nest-commander` يحوّل مزودات رف إلى أوامر قابلة للحقن. `CommandFactory.run` يبني السياق ويغلقه. الحزمة خارج نواة Nest. معرف المنظمة خيار محلول لا جسم طلب.

## ماذا بعد؟

ميزات المنصة التي لا يحتاجها المسار الأساسي: الترجمة والبريد والملفات و SSE و serverless والتطبيق الهجين.

## المصادر الرسمية

- NestJS — nest-commander: https://docs.nestjs.com/recipes/nest-commander
- NestJS — Standalone applications: https://docs.nestjs.com/standalone-applications
