# أساسيات OpenTelemetry

## الهدف من هذا الدرس

معرفة ماذا يوحّد OpenTelemetry لرف إن كانت المنظمة تجمع عدة لغات: أثر، ومقاييس، وسياق يمر بين HTTP وقاعدة.

## المشكلة التي يحلها

كل مكتبة تسجّل بطريقتها. الأثر ينقطع عند `fetch` إلى خدمة الإشعارات. فريق المنصة يريد نفس شكل span في Nest وفي عامل Python. حل خاص برف لا يُقرأ في المجمع العام.

## الفكرة ببساطة

OpenTelemetry API يصف spans ومقاييس وسياقًا يُحقن في الترويسات (`traceparent`). SDK يصدّر إلى مجمع. الإطار لا يختفي: ما زلتم تسمّون العمليات. المعيار يهم عندما تخرج الإشارة من Nest إلى أنظمة أخرى. إن كانت رف وحدها وتريدون اختصارًا بلا مجمع، الحزمة الرسمية في الدرس التالي خيار آخر لا يلغي هذا المعيار.

## التعريف التقني

المفاهيم: `Tracer` يبدأ span، والسياق النشط يتبع async، والـ exporter يرسل OTLP غالبًا. التلقين التلقائي (auto-instrumentation) يغطي HTTP و Prisma/pg إن رُكّبت المكتبات. span يدوي حول «نشر كتاب» يربط أبناء HTTP و SQL. المعرّف في السجل يجب أن يكون `traceId` حتى تجدوا الشجرة من سطر واحد. لا تسجّلوا جسم البطاقة داخل attributes. صفحة Nest للمراقبة تقول إن `@nestjs/observe` ليس بديلًا عن OpenTelemetry في بيئة متعددة اللغات.

## Mental Model

```text
طلب → span HTTP
        └ span publish
             └ span SQL
traceparent إلى خدمة الإشعارات
```

## أبسط مثال

```ts
import { trace } from '@opentelemetry/api';

export async function withPublishSpan<T>(bookId: string, run: () => Promise<T>) {
  return trace.getTracer('raf').startActiveSpan('books.publish', async (span) => {
    span.setAttribute('book.id', bookId);
    try {
      return await run();
    } finally {
      span.end();
    }
  });
}
```

## شرح المثال

- الاسم `books.publish` ثابت للبحث.
- `book.id` صفة منخفضة الحساسية.
- `finally` يغلق span حتى عند الرمي، وإلا يبقى الأثر مفتوحًا.

## ماذا يحدث داخليًا؟

1. SDK يبدأ مع العملية قبل استقبال الطلبات حتى لا يفوت الإقلاع.
2. جهاز HTTP يفتح span الجذر.
3. `startActiveSpan` يجعل الأبناء يتبعون السياق عبر `await`.
4. عميل HTTP الصادر يضيف `traceparent` إن كان ملقّنًا.
5. المجمّع يجمع النسخ. من غير مصدر، الاستدعاءات المحلية لا تظهر في لوحة.

## مثال عملي

```ts
export function logLine(msg: string) {
  const span = trace.getActiveSpan();
  const traceId = span?.spanContext().traceId;
  return { msg, traceId };
}
```

السطر يحمل `traceId` فتنتقلون من السجل إلى الشجرة. إن لم يوجد span فالطلب لم يُلقَّن؛ لا تخترعوا معرّفًا وتسموه trace.

## أخطاء شائعة

- تشغيل SDK بعد `listen` فيضيع أول المسارات.
- صفات فيها بريد أو جسم ويب هوك.
- اعتبار المكتبة بديلًا عن مقياس عدّ الأخطاء إن لم تصدّروا metrics أيضًا.
- توقع أن Nest يحقن OpenTelemetry لأنكم ثبّتم الحزمة فقط. يحتاج إقلاعًا وتلقينًا.

## متى نستخدمه؟

عندما تُجمع الإشارات مع خدمات غير Nest أو مع مجمع OTLP قائم في المنظمة.

## متى لا نستخدمه؟

لا تبنوا مجمعًا في الأسبوع الأول لمسار واحد إن كان `@nestjs/observe` يغطي سؤالكم. ولا تلقّنوا كل دالة صغيرة فتصبح الشجرة ضجيجًا.

## مقارنة

| | سجلات فقط | OpenTelemetry |
| --- | --- | --- |
| عبر الخدمات | تبحثون يدويًا | `traceparent` |
| المعيار | خاص | API مشترك |
| التشغيل | أخف | SDK ومجمّع |

## تمارين

### تمرين سهل

ماذا تضعون في السجل لفتح الشجرة؟

<details>
<summary>الحل</summary>

`traceId` من `spanContext` النشط، لا رقمًا عشوائيًا منفصلًا إن كان الأثر ملقّنًا.

</details>

### تمرين متوسط

لماذا تُغلق span في `finally`؟

<details>
<summary>الحل</summary>

حتى يُرسل المدة عند الاستثناء أيضًا. الترك يترك أثرًا بلا نهاية أو يسرب السياق.

</details>

### تمرين متقدم

خدمة الإشعارات Python لا ترى الأب. ماذا ينقص HTTP الصادر؟

<details>
<summary>الحل</summary>

حقن `traceparent` من السياق النشط عبر تلقين العميل أو ترويسة صريحة. من غيرها تبدأ شجرة جديدة.

</details>

## أسئلة مراجعة

- ماذا يوحّد المعيار؟
- لماذا SDK قبل `listen`؟
- أي صفة لا تُوضع على span؟
- متى لا يكون هذا أقصر طريق لرف وحدها؟

## الخلاصة

OpenTelemetry يربط spans وسياقًا عابرًا للخدمات بمعرّف واحد. يحتاج إقلاعًا وتصديرًا. هو الخيار عندما تعيش رف وسط أنظمة أخرى.

## ماذا بعد؟

إن أردتم إشارات بأسماء متحكمات Nest من غير مجمع، الدرس التالي الحزمة الرسمية.

## المصادر الرسمية

- OpenTelemetry — JavaScript: https://opentelemetry.io/docs/languages/js/
- NestJS — Observability overview: https://docs.nestjs.com/observability/overview
