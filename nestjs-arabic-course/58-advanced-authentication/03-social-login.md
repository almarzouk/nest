# Social Login

## الهدف من هذا الدرس

قرار رف عندما يعود شخص من مزود اجتماعي: إنشاء حساب، أو دخول حساب مربوط، أو رفض الربط بالبريد إن لم يُثبت الطرفان العنوان.

## المشكلة التي يحلها

أول زيارة بـ Google تنشئ مستخدمًا. الثانية يجب أن تدخل نفس المستخدم لا أن تنشئ صفًا ثانيًا. زيارة ثالثة تحمل بريد موظف موجود بكلمة مرور لم يُتحقق منه، فإن ربطتم فورًا بالبريد ملك المهاجم الحسابين معًا (pre-hijacking).

## الفكرة ببساطة

جدول هويات `(provider, subject) → userId`. العائد يجد الصف ويدخل. الجديد يُنشأ فقط إن كان البريد مثبتًا عند المزود، ويُربط فورًا. إن وُجد حساب محلي بنفس البريد، لا تربطوا إلا إذا كان بريدكم المحلي مثبتًا أيضًا. خلاف ذلك اطلبوا الدخول بكلمة المرور ثم الربط من جلسة مفتوحة.

## التعريف التقني

الحزمة تسأل `OidcAccountResolver.resolveUser(profile, tokens, context)`. `profile.subject` و `profile.provider` و `profile.email` و `profile.emailVerified` هي المعطيات. إرجاع `null` يرفض الدخول (403 في وصف الصفحة عندما لا يُثبت البريد). `linkTo` في السياق يعني أن الجلسة مفتوحة والشخص يريد ربط المزود بحسابه الحالي (`?link=true`). إن كان `subject` مربوطًا بشخص آخر فالنتيجة `ConflictException`. تسجيل المعالج عبر `AuthenticationRegistry.registerHandler('oidc', this)`. هذا قرار تطبيق تكتبونه. Passport strategy ما زالت طريقًا آخر لنفس القرار إن كانت هي مسار المشروع كله، لا طبقة ثانية بجانب المعالج.

## Mental Model

```text
موجود (provider, sub)     → المستخدم المربوط
لا، وبريد المزود غير مثبت → ارفض
لا حساب محلي              → أنشئ واربط
حساب محلي بريده غير مثبت  → ارفض واطلب كلمة المرور ثم ربطًا من الجلسة
حساب محلي بريده مثبت      → اربط sub به
linkTo وحساب آخر يملك sub → 409
```

## أبسط مثال

```ts
export async function decide(input: {
  linkedUserId: string | null;
  emailVerifiedByProvider: boolean;
  localEmailVerified: boolean | null;
}) {
  if (input.linkedUserId) return { action: 'sign-in' as const, userId: input.linkedUserId };
  if (!input.emailVerifiedByProvider) return { action: 'reject' as const };
  if (input.localEmailVerified === null) return { action: 'create' as const };
  if (!input.localEmailVerified) return { action: 'reject' as const };
  return { action: 'link' as const };
}
```

## شرح المثال

- `localEmailVerified === null` يعني لا صف بهذا البريد.
- الرفض عند بريد محلي غير مثبت يمنع من سجّل البريد بكلمة مرور قبل صاحب Google.
- الدالة لا تلمس HTTP. الحزمة تحوّل `null` من `resolveUser` إلى رفض.

## ماذا يحدث داخليًا؟

1. `finish` يتحقق من الرمز ثم يستدعي `resolveUser`.
2. البحث بالمفتاح `provider:subject`.
3. عند `linkTo` لا يُستخدم تطابق البريد. الجلسة الحالية هي الدليل، ويُفحص أن `sub` ليس لشخص آخر.
4. عند الإنشاء يُحفظ المستخدم ثم سطر الهوية.
5. الدخول يتم عبر `SignInService` في الحزمة، فمسار العامل الثنائي إن وُجد ما زال يُطبَّق.

## مثال عملي

```ts
import { ConflictException, Injectable } from '@nestjs/common';
import { AuthenticationRegistry, OidcAccountResolver, type OidcProfile, type OAuthTokens, type OidcResolveContext } from '@nestjs/authentication';

@Injectable()
export class ShelfAccountLinker extends OidcAccountResolver {
  constructor(registry: AuthenticationRegistry, private readonly users: {
    findByIdentity: (p: string, s: string) => Promise<{ id: string } | null>;
    findByEmail: (e: string) => Promise<{ id: string; emailVerified: boolean } | null>;
    link: (id: string, p: string, s: string) => Promise<{ id: string }>;
    create: (email: string) => Promise<{ id: string }>;
  }) {
    super();
    registry.registerHandler('oidc', this);
  }

  async resolveUser(profile: OidcProfile, _tokens: OAuthTokens, { linkTo }: OidcResolveContext) {
    const linked = await this.users.findByIdentity(profile.provider, profile.subject);
    if (linkTo) {
      if (linked && linked.id !== linkTo.id) throw new ConflictException('identity belongs to someone else');
      return this.users.link(linkTo.id, profile.provider, profile.subject);
    }
    if (linked) return linked;
    if (!profile.email || !profile.emailVerified) return null;
    const existing = await this.users.findByEmail(profile.email);
    if (!existing) {
      const user = await this.users.create(profile.email);
      return this.users.link(user.id, profile.provider, profile.subject);
    }
    if (!existing.emailVerified) return null;
    return this.users.link(existing.id, profile.provider, profile.subject);
  }
}
```

## أخطاء شائعة

- إنشاء مستخدم جديد في كل دخول لأن البحث كان بالبريد بعد ما تغيّر.
- ربط فوري بالبريد غير المثبت محليًا.
- السماح بـ `link=true` من صفحة موقع آخر. الحزمة تقيد الربط بجلسة قائمة وتعيد 403 لبدء الربط من غير ذلك السياق.
- تخزين access token الخاص بالمزود في `localStorage` واعتباره هوية رف.

## متى نستخدمه؟

عند أي مزود OIDC اجتماعي: Google في الدرس التالي أو غيره بنفس المعالج إن أضفتهم في `providers`.

## متى لا نستخدمه؟

لحسابات خدمة بلا مستخدم بشري. ولا لتجاوز العامل الثنائي: `SignInService` يكمل سياسة الحساب بعد الربط.

## مقارنة

| الحالة | النتيجة |
| --- | --- |
| `sub` معروف | دخول |
| بريد مزود غير مثبت | رفض |
| بريد محلي غير مثبت | رفض حتى يدخل بكلمة المرور ويربط من الجلسة |
| ربط والـ `sub` لغير هذا المستخدم | 409 |

## تمارين

### تمرين سهل

دخول ثانٍ بنفس Google. هل يُنشأ صف مستخدم؟

<details>
<summary>الحل</summary>

لا. `findByIdentity` يجد الربط ويعيد المستخدم نفسه.

</details>

### تمرين متوسط

شخص سجّل `reader@raf.example` بكلمة مرور ولم يفتح رسالة التحقق، ثم دخل بـ Google لنفس البريد. ماذا تعيد `resolveUser`؟

<details>
<summary>الحل</summary>

`null` لأن `emailVerified` المحلي غير صحيح. الصفحة تسمي هذا منع pre-hijacking. الدخول بكلمة المرور ثم «اربط Google» من الجلسة يستخدم `linkTo`.

</details>

### تمرين متقدم

لماذا ربط الجلسة لا يحتاج تطابق بريد؟

<details>
<summary>الحل</summary>

لأن الدليل هو الجلسة القائمة لا البريد. المستخدم أثبت أنه صاحب حساب رف، ويطلب إرفاق `sub` به. الشرط الوحيد أن هذا `sub` ليس مربوطًا بحساب آخر.

</details>

## أسئلة مراجعة

- ما مفتاح جدول الهويات؟
- متى يُرفض تطابق البريد؟
- ماذا يعني `linkTo`؟
- لماذا لا يُنشأ مستخدم عند كل عودة من المزود؟

## الخلاصة

تسجيل الدخول الاجتماعي قرار ربط: `sub` أولًا، ثم إنشاء أو ربط ببريد مثبت من الجهتين، والرفض عند الشك. الربط من جلسة مفتوحة لا يعتمد على البريد ويعارض إن كان `sub` لشخص آخر.

## ماذا بعد؟

القرار عام. الدرس التالي يملأ مزود Google: المسارات، `google()`، والكعكة.

## المصادر الرسمية

- NestJS — Authentication: https://docs.nestjs.com/security/authentication
