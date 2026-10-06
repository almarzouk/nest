# Token Revocation

## الهدف من هذا الدرس

إنهاء قدرة إثبات في رف فورًا: عائلة refresh لجهاز، أو جلسة متصفح، من غير انتظار انتهاء access token القصير وحده.

## المشكلة التي يحلها

القارئ فقد هاتفه. التدوير لن يحدث إن كان السارق هو من يجدد. انتظار ربع ساعة على access token الحالي، وأسبوعين على refresh إن لم يُستبدل، نافذة مفتوحة. زر «خروج من كل الأجهزة» يجب أن يسقط العائلة والجلسات الآن.

## الفكرة ببساطة

الإبطال علامة على الخادم. `TokenService.revoke` ينهي عائلة رمز التحديث: الرمز وأسلافه وخلفاءه. `SessionService.revoke` ينهي جلسة واحدة بشرط أن تكون لنفس المستخدم. خروج الكل ينهي جلسات الحساب ورموزه. access token الموقع قد يبقى حتى `exp` لأن التحقق بالتوقيع لا يسأل الجدول كل مرة؛ لذلك إبقاؤه قصيرًا جزء من الإبطال. لا تبنوا قائمة إبطال ضخمة لكل access إن كان عمره دقائق، إلا لحادثة تطلبون فيها رفضًا فوريًا أعقد.

## التعريف التقني

صفحة المصادقة تعرض `POST /auth/token/revoke` يستدعي `tokenService.revoke(refreshToken)` فتنتهي العائلة. الجلسات: `sessionService.revoke(id, { userId })` تعيد false لجلسة ليست لهذا المستخدم حتى لا ينهي معرّف في المسار جلسة شخص آخر، والرد عندها 404. معرّفات الجلسة المعروضة لصاحبها تجزئات ولا يُشتق منها سر الكعكة. مخزنكم ينفّذ `revokeRefreshTokenFamily` بعلامة `revoked` على صفوف العائلة، لا بحذف صامت يُعيد إدخال خلف خاسر في سباق. Passport ما زال موثّقًا كوصفة؛ إبطال جلسته هناك آلية تلك الوصفة. مسار رف الواحد لا يخلط المخزنين.

## Mental Model

```text
جهاز واحد: revoke(refresh) → العائلة ميتة
جلسة متصفح: revoke(sessionId, userId)
كل الأجهزة: جلسات الحساب + كل عائلاته
access الحالي: يموت عند exp أو عند فحص إضافي إن فرضتموه
```

## أبسط مثال

```ts
import { Body, Controller, HttpCode, Post } from '@nestjs/common';
import { TokenService } from '@nestjs/authentication';

@Controller('auth/token')
export class TokenRevocationController {
  constructor(private readonly tokens: TokenService) {}

  @Post('revoke')
  @HttpCode(200)
  async revoke(@Body('refreshToken') refreshToken: string) {
    await this.tokens.revoke(refreshToken);
    return { revoked: true };
  }
}
```

## شرح المثال

- المسار يحتاج إثباتًا أو يقبل الرمز نفسه كدليل الجهاز الذي يخرج، حسب ما تعرضه الوحدة. لا تسجّلوا الرمز.
- `{ revoked: true }` لا يعني أن access token نُزع من ذاكرة الجوال. الجوال يحذفه محليًا أيضًا.
- 200 بعد الإبطال المتكرر أسلم من 500. الإبطال الناجح مرتين ما زال «العائلة ميتة».

## ماذا يحدث داخليًا؟

1. تُعلَّم صفوف `familyId` بـ `revoked`.
2. أي `refresh` لاحق على فرد من العائلة يفشل بـ 401.
3. access token وُقّع قبل الإبطال يمر من فحص التوقيع حتى `exp` إن لم يوجد فحص عائلة على كل طلب.
4. لذلك `ttl` القصير من درس التدوير يحد المتبقي.
5. جلسة الكعكة مسار آخر: حذف صف الجلسة يجعل الطلب التالي بلا مستخدم حتى لو بقيت الكعكة في المتصفح.

## مثال عملي

خروج جهاز لا يخرج جارًا:

```ts
import { Controller, Delete, NotFoundException, Param } from '@nestjs/common';
import { CurrentUser, SessionService } from '@nestjs/authentication';

@Controller('auth/sessions')
export class SessionsController {
  constructor(private readonly sessions: SessionService) {}

  @Delete(':id')
  async revoke(@CurrentUser('id') userId: string, @Param('id') id: string) {
    const ok = await this.sessions.revoke(id, { userId });
    if (!ok) throw new NotFoundException();
  }
}
```

`userId` من الجلسة الحالية لا من الجسم. جسم `{ userId }` الذي يرسله العميل لا يُستخدم لتحديد الضحية. هذا نفس الحذر الذي سترونه مع المستأجر لاحقًا: المعرّف الحساس من الإثبات لا من الحقل الحر.

## أخطاء شائعة

- حذف refresh من الجوال فقط وتركه حيًا على الخادم.
- `revoke` بلا `userId` على الجلسات فيُنهى أي معرّف يُخمَّن.
- اعتبار الإبطال بديلًا عن `ttl` قصير لـ access token.
- إبطال عائلة عند كل 401 عادي فيُخرج المستخدم كلما انتهت مهلة شبكة.

## متى نستخدمه؟

فقدان جهاز، زر خروج، تغيير كلمة المرور (الصفحة أصلًا تنهي الجلسات ورموز التحديث عند إعادة التعيين)، واشتباه إعادة استخدام من درس التدوير.

## متى لا نستخدمه؟

عند كل طلب API. التعليم على العائلة يتم عند الخروج والحادثة لا عند القراءة. ولا كحذف مستخدم من الجدول؛ الإبطال يسقط الإثبات والصف قرار آخر.

## مقارنة

| الإجراء | ماذا يموت فورًا | ماذا قد يتبقى |
| --- | --- | --- |
| `tokenService.revoke` | عائلة refresh | access حتى `exp` |
| `sessionService.revoke` | تلك الجلسة | جلسات الأجهزة الأخرى |
| خروج الكل | جلسات الحساب والعائلات | access قصير حتى `exp` |

## تمارين

### تمرين سهل

بعد `POST /auth/token/revoke`، هل ينجح `refresh` بنفس الرمز؟

<details>
<summary>الحل</summary>

لا. العائلة أُبطلت، والرد 401.

</details>

### تمرين متوسط

لماذا تمرير `userId` من جسم الطلب إلى `revoke` خطر؟

<details>
<summary>الحل</summary>

لأن أي عميل يستطيع وضع معرّف شخص آخر. الصفحة تمرر مستخدم الجلسة وتطلب أن تطابق الجلسة صاحبها، وإلا 404.

</details>

### تمرين متقدم

سارق يملك access token حيًا لعشر دقائق بلا refresh. ماذا يحد نافذته اليوم، وما الذي لا يفعله `revoke` على عائلة لا يملكها أصلًا؟

<details>
<summary>الحل</summary>

`ttl` هو الحد إن لم تفحصوا العائلة عند كل access. `revoke` على refresh لا يسقط JWT الموقع قبل أن ينتهي. الخروج من كل الأجهزة يجب أن يسقط العائلات حتى لا يجدد، ويقصّر المتبقي عمر access. إن كانت الحادثة تستدعي قطعًا فوريًا للـ access، فذلك فحص إضافي (نسخة رمز أو قائمة قصيرة) وليس الوعد الافتراضي لـ JWT موقع.

</details>

## أسئلة مراجعة

- ماذا تنهي `TokenService.revoke`؟
- لماذا تُمرَّر هوية الجلسة لا هوية الجسم؟
- ماذا يبقى بعد إبطال refresh؟
- متى تُبطل العائلة تلقائيًا من غير زر خروج؟

## الخلاصة

الإبطال علامة خادم على عائلة الرمز أو على جلسة صاحبها. الجوال يحذف نسخه محليًا. access token القصير يحد ما يتبقى بعد الإبطال. المعرّف الذي يحدد الضحية يأتي من الإثبات.

## ماذا بعد؟

الحساب صار يعرف صاحبه. الفصل التالي يفصل بيانات مكتبة عن مكتبة أخرى على نفس العملية.

## المصادر الرسمية

- NestJS — Authentication: https://docs.nestjs.com/security/authentication
- NestJS — Passport: https://docs.nestjs.com/recipes/passport
