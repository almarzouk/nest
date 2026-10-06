# Request Logging

## الهدف من هذا الدرس

تسجيل كل طلب وارد بسطر واحد فيه الطريقة والمسار ورمز الحالة والمدة، بلا جسم التسجيل أو الترويسة السرية.

## المشكلة التي يحلها

سجلات الخدمات تقول «نُشر 42» ولا تقول إن `POST /books/42/publish` أعاد `403` في 12ms أو `201` في 800ms. من غير سطر الطلب لا ترى المسارات الصامتة التي لا تصل إلى `log` داخل الخدمة، مثل فشل أنبوب التحقق.

## الفكرة ببساطة

معترض بعد المسار يقيس الوقت من دخوله حتى اكتمال الرد. `tap` من RxJS يسجّل عند `next` وعند `error` من غير تغيير الجسم. لا تقرأ `body` كاملًا. المسار والمعرفات من `req` كافية. `StreamableFile` يُسجَّل كتنزيل لا كـ JSON.

## التعريف التقني

```ts
@Injectable()
export class RequestLogInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    if (context.getType() !== 'http') return next.handle();
    const http = context.switchToHttp();
    const req = http.getRequest<{ method: string; originalUrl: string }>();
    const started = Date.now();
    return next.handle().pipe(
      tap({
        next: () => this.finish(req, http.getResponse<{ statusCode: number }>().statusCode, started),
        error: () => this.finish(req, http.getResponse<{ statusCode: number }>().statusCode, started),
      }),
    );
  }

  private finish(req: { method: string; originalUrl: string }, statusCode: number, started: number) {
    this.logger.log({
      message: 'request',
      method: req.method,
      path: req.originalUrl.split('?')[0],
      statusCode,
      durationMs: Date.now() - started,
    });
  }
}
```

قص الاستعلام من `originalUrl` حتى لا يدخل `?token=` في السجل إن أخطأ عميل ووضع سرًا في الاستعلام. لا تسجّل `req.body`.

ركّب المعترض على التطبيق لا على مسار واحد حتى ترى `404` أيضًا. إن كان الرمز عند `error` ما زال `200` لأن المرشح لم يكتب بعد، سجّل `statusCode` داخل مرشح استثناءات بعد `status`. للتمرين، المعترض يكفي للنجاح، والمرشح يسجّل الفشل إن سبقه الرد.

## Mental Model

```text
دخول intercept
  next.handle()
    tap next أو error
      سطر method path status durationMs
```

## أبسط مثال

سطر ناجح:

```json
{"level":"log","context":"HTTP","message":"request","method":"POST","path":"/books/42/publish","statusCode":201,"durationMs":40}
```

سطر تحقق:

```json
{"message":"request","method":"POST","path":"/books","statusCode":400,"durationMs":3}
```

الثاني قد يأتي من المرشح إن لم يتحدث رمز الاستجابة لحظة `tap.error`. اقبل مصدرًا واحدًا حتى لا يتكرر السطر مرتين للطلب نفسه.

## شرح المثال

`getType() !== 'http'` يترك GraphQL والمقبس لمعترضاتهم إن احتجتها لاحقًا. خلط `originalUrl` لطلب HTTP مع سياق `ws` يعطي قيمًا فارغة. المدة تشمل الخدمة والطابور فقط إن انتظرت `add`، وهو ما تريده: بطء Redis يظهر في `durationMs`. المدة لا تشمل القص في العامل. ذلك سطر `process` منفصل.

## ماذا يحدث داخليًا؟

المعترض يغلّف التيار كما في فصل RxJS. `tap` لا يغيّر القيمة، لذلك `StreamableFile` يكمل. تسجيل `data` في `tap` قد يقرأ الملف أو يطبع كتابًا كاملًا. لذلك `finish` لا يأخذ `data`. رمز `201` يأتي من إطار Nest بعد المعالج. إن غيّرت الرمز في مرشح لاحق، قد يسجّل المعترض الرقم قبل التعديل. للتوحيد، مرشح الاستثناء يسجّل حالات الخطأ والمعترض حالات النجاح فقط، بشرط ألا يسجلا معًا على المسار السعيد.

## مثال عملي

استثناءات لا تُسجَّل كأجسام:

| المسار | ماذا في السطر |
| --- | --- |
| `POST /auth/login` | `path` و `statusCode` فقط |
| `POST /books/:id/cover` | المسار والرمز والمدة، لا حجم الملف ولا `originalname` |
| `GET /books/42/cover` | `statusCode` 200 وعبارة `stream` إن شئت، لا البايتات |

`originalname` قد يحتوي بيانات شخص. تجاهله.

## أخطاء شائعة

- تسجيل `req.body` على مسار الدخول.
- الإبقاء على `?access_token` في `path`.
- `console.log` ثانٍ داخل المعترض فيتكرر كل طلب.
- اعتبار مدة الطلب مدة القص في BullMQ.

## متى نستخدمه؟

على HTTP الوارد دائمًا في الإنتاج، بمستوى `log` للنجاح و `warn` إن زاد `durationMs` عن حد تضعه للمسار.

## متى لا نستخدمه؟

على كل حدث Socket.IO إن كان ذلك يملأ المجمع. هناك سطر اتصال وقطع يكفي. ولا تسجّل صحة Redis كل 30 ثانية بهذا المعترض. تلك نبضة `@Interval`.

## مقارنة

| المكان | ماذا يقيس |
| --- | --- |
| المعترض | الطلب الوارد حتى الرد |
| `IsbnService` | نداء الناشر |
| `CoversConsumer` | مهمة القص |

## تمارين

### تمرين سهل

أي أربعة حقول تصف الطلب في السطر؟

<details>
<summary>الحل</summary>
`method` و `path` و `statusCode` و `durationMs`.
</details>

### تمرين متوسط

لماذا يُقص الاستعلام من `originalUrl`؟

<details>
<summary>الحل</summary>
حتى لا يُحفظ سر وُضع في الاستعلام مثل رمز دخول.
</details>

### تمرين متقدم

لماذا لا يقيس هذا السطر زمن `resize` في العامل؟

<details>
<summary>الحل</summary>
لأن الرد يعود بعد `add` لا بعد القص. زمن العامل سطر داخل `process`.
</details>

## أسئلة مراجعة

- لماذا `tap` لا `map`؟
- ماذا تفعل لسياق ليس `http`؟
- أي جسم لا يدخل السطر أبدًا؟

## الخلاصة

سطر طلب واحد: طريقة ومسار بلا استعلام، ورمز، ومدة. بلا جسم وبلا ترويسة. زمن الطابور بعد الرد سطر آخر.

## ماذا بعد؟

معرف يربط سطر الطلب بسطر الناشر وسطر المهمة.

## المصادر الرسمية

- NestJS — Interceptors: https://docs.nestjs.com/interceptors
- NestJS — Logger: https://docs.nestjs.com/techniques/logger
