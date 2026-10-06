# Status Codes و Redirects

## الهدف من هذا الدرس

ضبط رمز نجاح الاستجابة بـ `@HttpCode`، وتحويل العميل بـ `@Redirect`.

## المشكلة التي يحلها

`POST` الذي يستبدل قسيمة بلا جسم ما زال يرسل 201. ورابط الدليل القديم يحتاج 301 إلى عنوان جديد. الرمز جزء من عقد HTTP، والـ Decorator يعلنه بجانب المعالج.

## الفكرة ببساطة

نجاح Nest الافتراضي 200، وطريقة POST تستخدم 201. `@HttpCode(204)` يستبدل ذلك على المعالج. `@Redirect(url, status)` يرسل تحويلًا، وافتراض الرمز 302 إن حُذف. إعادة كائن `{ url, statusCode }` تستبدل وسائط الـ Decorator.

## التعريف التقني

```ts
@Post()
@HttpCode(204)
redeem() {
  return;
}
```

`HttpCode` و `HttpStatus` من `@nestjs/common`. `HttpStatus.NO_CONTENT` هو 204، و `HttpStatus.OK` هو 200.

عندما يعتمد الرمز على نتيجة متغيرة لخطأ، ارمِ استثناءً. Exception Filter يحوّله إلى استجابة، وذلك فصل لاحق. `@Res()` يقدر أن يكتب رمزًا أيضًا، وهو وضع المكتبة في الدرس التالي. للأسلوب القياسي استخدم `@HttpCode` عندما يكون الرمز ثابتًا على المعالج.

`@Redirect(url, statusCode)` : الرمز الافتراضي 302. القيمة المعادة من نوع `HttpRedirectResponse` تستبدل الوسائط:

```ts
return { url: 'https://docs.example.com/v5/', statusCode: 301 };
```

إن لم تُعد الدالة كائنًا، تبقى وسائط الـ Decorator.

## Mental Model

```text
POST /coupons/redeem     @HttpCode(204)  -> 204 بلا جسم متعمد
POST /coupons            بلا HttpCode    -> 201 والجسم المعاد
GET  /coupons/manual     @Redirect(...)  -> 302 أو ما تعيده الدالة
```

## أبسط مثال

```ts
import { Controller, HttpCode, HttpStatus, Post } from '@nestjs/common';
import { CouponsService } from './coupons.service.js';

@Controller('coupons')
export class CouponsController {
  constructor(private readonly coupons: CouponsService) {}

  @Post('redeem')
  @HttpCode(HttpStatus.NO_CONTENT)
  redeem() {
    this.coupons.redeemCurrent();
  }

  @Post('search')
  @HttpCode(HttpStatus.OK)
  search() {
    return this.coupons.searchActive();
  }
}
```

## شرح المثال

- `redeem` عملية POST بلا تمثيل جديد. 204 يطابق «تم، ولا جسم».
- `search` عملية POST لأن جسم التصفية قد يكبر لاحقًا، والعقد يتفق على 200. من غير `@HttpCode` كان الرمز سيصبح 201.
- `HttpStatus` أوضح من الرقم العاري عند القراءة، والرقم `204` مقبول أيضًا.
- `redeem` لا تعيد قيمة. مع 204 هذا مقصود.

## ماذا يحدث داخليًا؟

طبقة الاستجابة تختار 201 لمُعالج `@Post` و 200 لغيره، ثم تطبق `@HttpCode` إن وُجد. `@Redirect` يملأ عنوان `Location` والرمز. إن أعادت الدالة `url` أو `statusCode` تُستبدل القيم المطابقة من الـ Decorator وتُحفظ البقية.

الاستثناء الذي ترميه الخدمة لا يمر على `@HttpCode`. الرمز عندئذٍ يأتي من طبقة الاستثناءات، لا من رمز النجاح.

## مثال عملي

```ts
import { Controller, Get, Query, Redirect } from '@nestjs/common';

@Controller('coupons')
export class CouponDocsController {
  @Get('manual')
  @Redirect('https://docs.example.com/coupons', 302)
  manual(@Query('edition') edition?: string) {
    if (edition === '2024') {
      return { url: 'https://docs.example.com/coupons/2024', statusCode: 301 };
    }
  }
}
```

`GET /coupons/manual` يحوّل بـ 302 إلى الدليل العام. `?edition=2024` يعيد كائنًا يستبدل العنوان والرمز إلى 301. الاستعلام هنا شرط للتحويل، وقراءة `@Query` نفسها مشروحة في درسها.

## أخطاء شائعة

| الخطأ | الأثر |
| --- | --- |
| توقع 200 من `@Post` الذي يعيد كائنًا | الافتراضي 201 حتى يُضاف `@HttpCode(200)` |
| جسم JSON مع `@HttpCode(204)` | العقد يقول «بلا محتوى» والجسم يربك العميل |
| `@Redirect` مع نسيان أن الكائن المعاد يستبدل الوسائط | طبعة 2024 تبقى على عنوان 302 العام |
| رمي خطأ بأسلوب `@HttpCode(404)` | رمز النجاح لا يصف الفشل؛ الفشل استثناء |

## متى نستخدمه؟

`@HttpCode` عندما يخالف الرمز الثابت الافتراض: 204 بعد تنفيذ، أو 200 على POST للبحث. `@Redirect` عندما يكون عنوان الاستجابة موقعًا آخر، ثابتًا أو حسب قيمة ترجعها الدالة.

## متى لا نستخدمه؟

لرمز يتغير مع كل خطأ مجال (رصيد، صلاحية). ذلك استثناء و Exception Filter. ولضبط رأس `Location` يدويًا مع باقي الأسلوب القياسي من غير تحويل جاهز؛ `@Redirect` يغطي حالة التحويل.

## مقارنة

| الأداة | النتيجة |
| --- | --- |
| بلا Decorator على GET | 200 مع القيمة المعادة |
| بلا Decorator على POST | 201 مع القيمة المعادة |
| `@HttpCode(status)` | يستبدل رمز النجاح لذلك المعالج |
| `@Redirect(url, status)` | تحويل، والافتراضي 302، والكائن المعاد يستبدل الوسائط |

## تمارين

### تمرين سهل

ما رمز `POST /coupons` الذي يعيد `{ id: 'c1' }` من غير `@HttpCode`؟

<details><summary>الحل</summary>

201.

</details>

### تمرين متوسط

كيف تجعل `POST /coupons/search` يعيد 200 مع قائمة JSON؟

<details><summary>الحل</summary>

`@HttpCode(200)` أو `@HttpCode(HttpStatus.OK)` على المعالج، وأعد القائمة من الدالة. Nest ترسل القائمة JSON لأن القيمة كائن أو مصفوفة.

</details>

### تمرين متقدم

`@Redirect('https://docs.example.com/coupons', 302)` والدالة تعيد `{ url: 'https://docs.example.com/coupons/2024' }` فقط. ما الرمز وما العنوان؟

<details><summary>الحل</summary>

العنوان يصبح نسخة 2024 لأن `url` في الكائن يستبدل وسيط الـ Decorator. الرمز يبقى 302 لأن الكائن لم يمرر `statusCode`.

</details>

## أسئلة مراجعة

- ما رمز النجاح الافتراضي لغير POST؟
- ما الرمز الافتراضي لـ `@Redirect` عند حذف الوسيط الثاني؟
- ماذا يستبدل الكائن المعاد من معالج `@Redirect`؟

## الخلاصة

200 و 201 هما افتراض Nest، و `@HttpCode` يثبّت غيرهما على المعالج. `@Redirect` يحوّل العميل، وإعادة `{ url, statusCode }` تخصّص التحويل لذلك الطلب.

## ماذا بعد؟

كائن الطلب الخام للمنصة: Request Object.

## المصادر الرسمية

- NestJS — Status code: https://docs.nestjs.com/controllers#status-code
- NestJS — Redirection: https://docs.nestjs.com/controllers#redirection
