# Error Handling

## الهدف من هذا الدرس

ترجمة فشل مورد المنشورات إلى رموز HTTP مفهومة: `400` و `404` و `409`، من غير تسريب أخطاء Prisma إلى العميل.

## المشكلة التي يحلها

خطأ قيد فريد يخرج كـ `500` مع تتبع فيه اسم الجدول. غياب المقال يخرج `200` مع `null`. العميل لا يفرّق بين «البريد مستخدم» و«القاعدة سقطت». كل متحكم يخترع شكل JSON مختلفًا.

## الفكرة ببساطة

الخدمات ترمي استثناءات HTTP من `@nestjs/common` للحالات المتوقعة. مرشح الاستثناءات المدمج يبني الجسم `{ statusCode, message, error }`. ما ليس متوقعًا، مثل انقطاع PostgreSQL، يبقى `500` بلا تفاصيل داخلية في الإنتاج. ترجمة رموز Prisma تحدث في الخدمة أو في مرشح ضيق يعرف `PrismaClientKnownRequestError`.

## التعريف التقني

| الحالة | الاستثناء | مثال في المشروع |
| --- | --- | --- |
| جسم غير صالح | يرميه `ValidationPipe` | عنوان فارغ |
| معامل غير عددي | `ParseIntPipe` | `/posts/abc` |
| صف غائب | `NotFoundException` | مستخدم أو مقال أو أب التعليق |
| قيد فرادة | `ConflictException` | بريد مكرر `P2002` |
| عطل غير متوقع | لا تمسكه كـ `400` | القاعدة متوقفة |

```ts
throw new NotFoundException("Post not found");
```

لا تبنِ `res.status(404).json(...)` في كل مسار. الاستثناء يمر في المرشح نفسه الذي يمر فيه فشل الأنبوب، فيبقى الشكل واحدًا.

رمز Prisma `P2002` تكرار، و `P2003` مفتاح أجنبي، و `P2025` تحديث أو حذف صف غير موجود. ترجم ما وعدت به في العقد، واترك الباقي يسقط كـ `500` حتى تراه في السجل.

رسالة العميل ثابتة مثل `"Email already registered"`. لا تضمّن SQL ولا قيمة البريد كاملة إن كان ذلك يكشف حسابات في مسار عام. في هذا الفصل المسار مفتوح أصلًا، والعادة تبقى.

## Mental Model

```text
فشل متوقع في الخدمة -> HttpException -> مرشح مدمج -> JSON ورمز
فشل Prisma معروف   -> ترجمة في الخدمة -> نفس المرشح
فشل مجهول          -> 500 وسجل على الخادم
```

## أبسط مثال

```ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpStatus } from "@nestjs/common";
import { Prisma } from "../generated/prisma/client.js";
import { Response } from "express";

@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  catch(exception: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse<Response>();
    if (exception.code === "P2002") {
      response.status(HttpStatus.CONFLICT).json({
        statusCode: HttpStatus.CONFLICT,
        message: "Unique value already exists",
        error: "Conflict",
      });
      return;
    }
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
      message: "Internal server error",
      error: "Internal Server Error",
    });
  }
}
```

## شرح المثال

الفلتر شبكة أمان للرمز الذي نسيته الخدمة. الأفضل ترجمة `P2002` بجانب إنشاء المستخدم حيث تعرف أن الحقل هو البريد، والرسالة أدق. الفلتر العام يقول «قيمة فريدة موجودة» فقط. سجّل `exception.code` على الخادم، لا في JSON.

تسجيل الفلتر بـ `@Catch` على متحكم يضيّقه. تسجيله عامًا يغطّي كل الموارد. أبقِ الرسائل الخاصة في الخدمات.

## ماذا يحدث داخليًا؟

الاستثناء غير الممسوك في المسار يصل إلى طبقة الفلاتر، الأخص أولًا ثم الأعم. `HttpException` يعرفه الفلتر المدمج ويستخرج `getStatus()` و `getResponse()`. فلترك إن مسك الخطأ أولًا فهو المسؤول عن الجسم. لا تكتب الرأس مرتين.

انقطاع الاتصال ليس `KnownRequestError` بالضرورة. دعه `500`. أعد المحاولة ليست مسؤولية المتحكم.

## مثال عملي

جدول قصير تختبره يدويًا:

- `POST /users` ببريد مستخدم: `409` ورسالتك.
- `GET /posts/999`: `404` و `Post not found`.
- `POST /posts` بعنوان فارغ: `400` من الأنبوب.
- أوقف PostgreSQL ثم `GET /posts/1`: `500` بلا مضيف القاعدة في الجسم.

اختبار e2e يثبت الرموز الثلاثة الأولى. لا تحتاج إلى إطفاء القاعدة في كل تشغيل.

## أخطاء شائعة

- `try/catch` فارغ يحوّل كل شيء إلى `200`.
- إرجاع تتبع Prisma للعميل.
- `404` لخطأ تحقق.
- شكل JSON مختلف في كل متحكم.

## متى نستخدمه؟

لكل فشل تعرفه عند كتابة المورد: غياب، تكرار، جسم سيئ.

## متى لا نستخدمه؟

لإخفاء عطل البنية تحت `400`. إن كانت القاعدة متوقفة فالرمز `500` حتى يتنبّه المشغّل.

## مقارنة

| استثناء HTTP في الخدمة | `res.status` في المتحكم |
| --- | --- |
| يمر بالفلتر الموحّد | شكل لكل مسار |
| يُختبر برمي الدالة | يحتاج HTTP كاملًا |
| يناسب `404` و `409` | يسهل نسيان مسار |

## تمارين

### تمرين سهل

أي رمز لبريد مكرر، وأي رمز لمقال غائب؟

<details>
<summary>الحل</summary>

`409` للتكرار، و `404` للغياب.

</details>

### تمرين متوسط

لماذا لا يكفي فلتر عام يقول «قيمة فريدة» عند إنشاء المستخدم؟

<details>
<summary>الحل</summary>

العميل لا يعرف إن كان البريد أو حقلًا آخر. الترجمة بجانب `UsersService.create` تسمي البريد. الفلتر العام يبقى احتياطًا.

</details>

### تمرين متقدم

`P2003` ظهر عند إنشاء تعليق. ماذا حدث، وما الرمز المناسب إن وعدت أن مسار أب غير موجود هو `404`؟

<details>
<summary>الحل</summary>

المفتاح الأجنبي رفض `postId`، غالبًا لأن المقال حُذف بعد الفحص أو لم يُفحص. إن كان العقد `404` للمقال الغائب، ترجم `P2003` على هذا المسار إلى `NotFoundException`. لا تتركه `500`.

</details>

## أسئلة مراجعة

- من يبني جسم `HttpException` افتراضيًا؟
- ماذا يُسجَّل على الخادم ولا يُرسل؟
- أي رمز لانقطاع القاعدة؟

## الخلاصة

الأخطاء المتوقعة استثناءات HTTP برموز ثابتة. رموز Prisma تُترجم حيث تعرف المعنى. العطل غير المتوقع `500` بلا تسريب.

## ماذا بعد؟

[Pagination](./07-pagination.md) يقطع قائمة المنشورات إلى صفحات.

## المصادر الرسمية

- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- NestJS — Prisma: https://docs.nestjs.com/data/prisma
