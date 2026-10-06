# ClassSerializerInterceptor

## الهدف من هذا الدرس

ربط `ClassSerializerInterceptor` حتى تُحذف الحقول المعلَّمة عند إرجاع نسخة صنف، ومعرفة متى يمر الكائن العادي بلا تغيير.

## المشكلة التي يحلها

`@Exclude()` على `passwordHash` موجود والصنف صحيح، والعميل ما زال يرى التجزئة. المعالج يعيد كائن Prisma العادي. التحذير في الصفحة: الكائن العادي، مثل `{ user: new UserEntity() }`، لا يُسلسَل كما تتوقع. الاعتراض يستدعي `instanceToPlain` على النسخ.

## الفكرة ببساطة

الاعتراض من `@nestjs/common`. على المسار أو المتحكم تستدعي `@UseInterceptors(ClassSerializerInterceptor)`. المعالج يعيد `new UserResponse(...)`. الحقول ذات `@Exclude` تسقط من JSON. إن كان الرجوع كائنًا عاديًا، مرّر `@SerializeOptions({ type: UserResponse })` ليحوّله الاعتراض إلى نسخة قبل التسلسل.

## التعريف التقني

```ts
import { Exclude } from "class-transformer";

export class UserResponse {
  id: number;
  email: string;
  role: string;

  @Exclude()
  passwordHash?: string;

  constructor(partial: Partial<UserResponse>) {
    Object.assign(this, partial);
  }
}
```

```ts
@UseInterceptors(ClassSerializerInterceptor)
@Get("me")
me(@CurrentUser() user: UserResponse): UserResponse {
  return new UserResponse(user);
}
```

الشكل من صفحة Nest، مع حقول المكتبة. الاستجابة عندها بلا `passwordHash`.

للاعتراض العام، الصفحة تشرح مسار المخطط مع `Reflector`. المسار نفسه لـ `ClassSerializerInterceptor`: إما `APP_INTERCEPTOR` فيحقن Nest الاعتمادات، أو `new ClassSerializerInterceptor(app.get(Reflector))` في `useGlobalInterceptors`. من غير `Reflector` لا تُقرأ `@SerializeOptions`.

```ts
@SerializeOptions({ type: UserResponse })
@Get("me")
me() {
  return this.users.findMe();
}
```

هنا `findMe` يعيد كائنًا عاديًا. الخيار `type` يستدعي التحويل إلى `UserResponse` ثم المزخرفات. الصفحة تفضّل إعلان نوع الإرجاع حتى يفحص TypeScript الشكل. `plainToInstance` وحدها لا تعطي ذلك الفحص.

`excludePrefixes: ['_']` في `@SerializeOptions` يمرَّر إلى `instanceToPlain` ويستبعد ما يبدأ بشرطة سفلية.

`@Transform(({ value }) => value.name)` يستبدل كائنًا باسمه، كما في مثال `role` في الصفحة.

اعتراض المخطط بديل: `@SerializeOptions({ schema: userResponseSchema })` مع `StandardSchemaSerializerInterceptor`. مصفوفة النتائج تأخذ مخطط العنصر لا مخطط المصفوفة. عدم التطابق `500`. هذا الدرس يبقى على الأصناف. المخطط مفيد حين تريد قائمة سماح تحذف أي مفتاح زائد حتى بلا `@Exclude`.

## Mental Model

```text
return new UserResponse(row)
  instanceToPlain
  @Exclude يسقط
return row العادي بلا type
  JSON.stringify ينشر التجزئة
```

## أبسط مثال

```ts
@SerializeOptions({ excludePrefixes: ["_"] })
@Get(":id")
findOne(): UserResponse {
  return new UserResponse({ id: 1, email: "ada@library.test", role: "reader" });
}
```

## شرح المثال

البادئة طبقة إضافية فوق `@Exclude`. حقل `_debug` يختفي. لا تضع السر في اسم بلا مزخرف ثم تعتمد على البادئة فقط إن نسيت الشرط. `@Exclude` على `passwordHash` أوضح للقارئ.

`ClassSerializerInterceptor` على الصنف يغطي `findOne` و `me`. مسار يعيد `{ user: new UserResponse() }` يسلسل الغلاف ككائن عادي وقد يضم النسخة بشكل لا يمر على القواعد كما تتوقع الصفحة. أعد النسخة من الجذر أو صفِّ الغلاف.

## ماذا يحدث داخليًا؟

بعد نجاح المعالج يقرأ الاعتراض البيانات الوصفية. `instanceToPlain` يمشي الخصائص. `@Exclude` يحذف. الناتج يذهب إلى محوّل JSON. الاستثناء الذي يُرمى قبل الإرجاع لا يمر على شكل النجاح. لا تضع الصف داخل `HttpException`.

Prisma لا يعيد `UserResponse`. `new UserResponse(row)` أو `type` هما الجسر. `Object.assign` ينسخ `passwordHash` إلى النسخة ثم يحذفه التسلسل. النسخة في الذاكرة قد تحمل السر أثناء الطلب. لا تسجّلها.

`StreamableFile` لتنزيل لا يُمرَّر على هذا التحويل.

## مثال عملي

اختبار طلب `GET /me` مع مستخدم فيه تجزئة يتوقع المفاتيح `id` و `email` و `role` فقط. اختبار ثانٍ يعيد الكائن العادي بلا `type` ويُظهر أن التجزئة تبقى، فيوثّق سبب الخيار. بعد إضافة `{ type: UserResponse }` يختفي الحقل. ثبّت الاعتراض على `UsersController` لا على مسار واحد ثم انسَ `PATCH`.

## أخطاء شائعة

- مزخرف `@Exclude` بلا اعتراض.
- إرجاع كائن Prisma بلا `type`.
- اعتراض عام بلا `Reflector` أو بلا `APP_INTERCEPTOR` فلا تُقرأ الخيارات.
- تغليف النسخة داخل مفتاح `user` وتوقع حذف التجزئة.

## متى نستخدمه؟

حين تبني الإخراج بأصناف `class-transformer` وتريد الحذف في مكان واحد.

## متى لا نستخدمه؟

على مسار يعيد نصًا خامًا أو ملفًا. ولا كوسيلة تمنع جلب العمود. `select` يبقي السر خارج العملية من الأساس.

## مقارنة

| `new UserResponse` | `@SerializeOptions({ type })` |
| --- | --- |
| النسخة في المعالج | التحويل داخل الاعتراض |
| صريح | يناسب رجوع Prisma العادي |
| يعمل مع الاعتراض | يحتاج قراءة البيانات الوصفية |

## تمارين

### تمرين سهل

من أي حزمة يُستورد `ClassSerializerInterceptor`؟

<details>
<summary>الحل</summary>

من `@nestjs/common`.

</details>

### تمرين متوسط

لماذا تبقى `passwordHash` مع `@Exclude` موجود؟

<details>
<summary>الحل</summary>

القيمة ليست نسخة الصنف، أو الاعتراض غير مربوط. أعد `new` أو اضبط `type`.

</details>

### تمرين متقدم

كيف تثبّت الاعتراض لكل التطبيق مع خيارات على المسارات؟

<details>
<summary>الحل</summary>

`APP_INTERCEPTOR` أو `new ClassSerializerInterceptor(app.get(Reflector))` حتى تُقرأ `@SerializeOptions`.

</details>

## أسئلة مراجعة

- ماذا يستدعي الاعتراض من `class-transformer`؟
- ماذا يفعل خيار `type`؟
- هل يُسلسَل `StreamableFile`؟

## الخلاصة

الاعتراض يطبّق المزخرفات على النسخ. الكائن العادي يحتاج `type` أو بناءً صريحًا. من غير الربط، `@Exclude` زينة على الصنف.

## ماذا بعد؟

[Exclude و Expose](./03-exclude-and-expose.md) الحذف والإظهار المحسوب.

## المصادر الرسمية

- NestJS — Serialization: https://docs.nestjs.com/application/serialization
