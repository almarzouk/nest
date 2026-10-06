# توثيق Auth

## الهدف من هذا الدرس

إظهار قفل الجلسة وقفل `Bearer` في Swagger بما يطابق `@nestjs/authentication` في هذه الدورة.

## المشكلة التي يحلها

`POST /posts` في الواجهة يُجرَّب بلا رأس فيأخذ `401`. القارئ يظن أن المسار معطوب. أو تُضاف مصادقة Basic لأن المثال الرسمي بدأ بها، بينما المكتبة لا تقبل Basic. وثيقة بلا مخطط أمن تخفي أن القراءة العامة مفتوحة والكتابة مغلقة.

## الفكرة ببساطة

`DocumentBuilder` يسجّل مخطط الأمن مرة. المزخرف على المتحكم أو العملية يطبّقه. `@ApiBearerAuth` مع `addBearerAuth` لرمز الوصول. `@ApiCookieAuth` مع `addCookieAuth` لكعكة الجلسة. اسم الكعكة في الوثيقة يطابق ما يضعه الخادم حتى يفهم المجرب أي قيمة يلصق. Swagger لا يسجّل الدخول عنك.

## التعريف التقني

من صفحة Security:

```ts
const config = new DocumentBuilder()
  .setTitle("Library posts")
  .setVersion("1.0")
  .addCookieAuth("__Host-sid")
  .addBearerAuth()
  .build();
```

```ts
import { ApiBearerAuth, ApiCookieAuth } from "@nestjs/swagger";

@ApiCookieAuth()
@Controller("posts")
export class PostsController {}
```

`addBearerAuth()` يعرّف مخطط HTTP من نوع `bearer` بالاسم الافتراضي الذي يربطه `@ApiBearerAuth()`. `addCookieAuth` يأخذ اسم الكعكة. في الحزمة الاسم `__Host-sid` كما في فصل الكعكات. المثال في الصفحة يستخدم `'optional-session-id'` كموضع لاسم تكتبه أنت.

`@ApiSecurity('basic')` مع `addSecurity` لمخطط تبنيه يدويًا. `@ApiBasicAuth` و `addBasicAuth` لمسار Basic. المكتبة لا تستخدمه للدخول. لا تضفه حتى لا يظهر قفل لا يقبله الحارس.

`GET` المنشورات العامة: لا تضع المزخرف على ذلك المعالج، أو أزل الأمن عن العملية إن ورث المتحكم القفل. القفل على الصنف يعم الطرق. القراءة العامة تستحق عملية بلا أمن حتى لا يطلب Swagger مفتاحًا لـ `GET /posts`.

تجربة الواجهة: Authorize ثم ألصق الرمز لـ Bearer. الكعكة أصعب من الواجهة لأن المتصفح يرفق `__Host-sid` حسب أصل الصفحة لا حسب حقل Authorize دائمًا. للتجربة اليدوية ابقَ على نفس الأصل، أو استخدم Bearer في Swagger والجوال، واترك الكعكة لمتصفح التطبيق. الوثيقتان معًا صادقتان لأن الحزمة تقبل الجلسة أولًا ثم `JwtBearerProvider` إن رُتّب.

Passport ما زال وصفة رسمية. إن وثّقت مشروعًا قديمًا على `AuthGuard('jwt')` فالقفل Bearer يكفي هناك. لا تعلن الاثنين على مسار واحد في التطبيق كما سبق في الاختيار.

## Mental Model

```text
addCookieAuth + addBearerAuth في الوثيقة
المسار المحمي يحمل قفلًا
GET العام بلا قفل
Authorize في الواجهة يملأ الرأس أو يشرح الكعكة
الحارس ما زال يرفض الطلب الفارغ
```

## أبسط مثال

```ts
@ApiBearerAuth()
@Post()
@ApiUnauthorizedResponse({ description: "Missing session or bearer" })
create(@Body() dto: CreatePostDto) {
  return this.posts.create(dto);
}
```

## شرح المثال

هذا القفل لمسار واحد إن كانت بقية المتحكم عامة. `401` في الاستجابات يشرح فشل التجربة. لا تكتب كلمة المرور في وصف المزخرف. الرمز الذي يُلصق في Authorize هو `accessToken` قصير العمر من `TokenService`، لا تجزئة كلمة المرور.

## ماذا يحدث داخليًا؟

المخططات تذهب إلى `components.securitySchemes`. العملية المحمية تأخذ مصفوفة `security`. Swagger UI يرسم قفلًا. عند التنفيذ يضيف `Authorization: Bearer` إن كان المخطط bearer ومُلئ. الكعكة تعتمد على المتصفح. الوثيقة لا تستدعي `SignInService`.

مسار `@Public()` مع قفل في الوثيقة يربك القارئ: الواجهة تطلب رمزًا والخادم يتجاهل غيابه. زامن `@Public` مع غياب المزخرف.

## مثال عملي

`AuthController` للدخول يبقى بلا قفل على `sign-in` لأنه `@Public`. `GET /me` يحمل `@ApiCookieAuth` و `@ApiBearerAuth` إن قبل الاثنين. بعد فتح `/api` تأكد أن `sign-in` بلا قفل وأن `POST /posts` بقفل. نفّذ إنشاءً بلا Authorize وتوقع `401` في الاستجابة لا صفحة Swagger فارغة.

لا تضع قيمة `__Host-sid` حقيقية في `example` داخل المستودع.

## أخطاء شائعة

- `addBasicAuth` لمشروع الجلسة.
- قفل على المتحكم كله بما فيه `GET` العام.
- اسم كعكة في الوثيقة غير `__Host-sid`.
- لصق `passwordHash` في خانة Bearer.

## متى نستخدمه؟

على كل عملية يغطيها الحارس، بمخطط يطابق ما يرسله العميل فعلًا.

## متى لا نستخدمه؟

كآلية دخول. Authorize يخزن الرمز في المتصفح لتجربة الواجهة. احذف التجربة على جهاز مشترك. الإنتاج الذي يعرض Swagger للعلن يعرض شكل العقد لا أسرار المستخدمين.

## مقارنة

| `@ApiCookieAuth` | `@ApiBearerAuth` |
| --- | --- |
| يوثّق `__Host-sid` | يوثّق `Authorization: Bearer` |
| مناسب لواجهة نفس الأصل | مناسب للجوال ولتجربة Swagger |
| يحتاج `addCookieAuth` | يحتاج `addBearerAuth` |

## تمارين

### تمرين سهل

أي مستدعٍ في `DocumentBuilder` يسجّل Bearer الجاهز؟

<details>
<summary>الحل</summary>

`addBearerAuth()`، والمزخرف `@ApiBearerAuth()`.

</details>

### تمرين متوسط

لماذا يبقى `POST /auth/sign-in` بلا `@ApiCookieAuth`؟

<details>
<summary>الحل</summary>

المسار عام قبل وجود جلسة. القفل يوحي أن الطلب يحتاج كعكة مسبقة.

</details>

### تمرين متقدم

المتحكم عليه `@ApiCookieAuth` و `GET /posts` عام. كيف تُصلح الوثيقة؟

<details>
<summary>الحل</summary>

انقل المزخرف إلى عمليات الكتابة فقط، واترك القراءة بلا أمن. أو أقسم المتحكم إن اختلط العام بالمحمي بكثرة.

</details>

## أسئلة مراجعة

- ما اسم الكعكة الذي يُمرَّر إلى `addCookieAuth` هنا؟
- هل الواجهة تسجّل الدخول؟
- أي مخطط لا نعلنه لمكتبة الدورة؟

## الخلاصة

الوثيقة تقفل ما يقفله الحارس: كعكة الجلسة و Bearer. التسجيل والقراءة العامة بلا قفل. Basic لا يدخل عقد هذه الدورة.

## ماذا بعد؟

[Examples و Response Types](./06-examples-and-responses.md) أجسام النجاح والقوائم.

## المصادر الرسمية

- NestJS — OpenAPI security: https://docs.nestjs.com/openapi/security
- NestJS — Authentication: https://docs.nestjs.com/security/authentication
