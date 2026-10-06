# CASL ومتى يناسب

## الهدف من هذا الدرس

معرفة CASL كمكتبة قدرات خارجية، ومتى لا تستبدل `@nestjs/authorization` في هذه الدورة.

## المشكلة التي يحلها

البحث عن «authorization nestjs» يعرض مقالات CASL قبل صفحة Nest الحالية. الفريق يثبّت المكتبة ثم يعيد كتابة `PostPolicy` مرتين. أو يتجنب الحزمة الرسمية لأنها لا تشبه `ability.can`.

## الفكرة ببساطة

CASL مكتبة JavaScript تبني كائن قدرة: أفعال على أنواع، وربما شروط على الحقول. هي ليست حزمة `@nestjs` في صفحة التفويض الحالية. الدورة تعتمد `@Policy` و `AuthorizationService`. CASL يناسب فريقًا يملك قواعدًا كثيرة متشابهة ويريد بناء القدرة من بيانات، أو مشروعًا قائمًا عليها أصلًا.

## التعريف التقني

نمط CASL الشائع، للتعرّف لا للاعتماد:

```ts
import { AbilityBuilder, createMongoAbility } from "@casl/ability";

export function buildFor(user: PublicUser | null) {
  const { can, build } = new AbilityBuilder(createMongoAbility);
  can("read", "Post");
  if (user?.roles.includes("editor")) {
    can("create", "Post");
  }
  if (user) {
    can("update", "Post", { authorId: user.id });
  }
  return build();
}
```

`can("update", "Post", { authorId })` شرط على شكل السجل، قريب من دالة `update(user, post)`. التقييم يتم عبر `ability.can("update", subject)`.

لا يوجد في هذا الدرس `CaslModule` رسمي داخل توثيق Nest الحالي الذي قرأناه. إن أضفت المكتبة فأنت تملك ربطها بحارس تكتبه. `@Can` من `@nestjs/authorization` لا يقرأ كائن CASL.

اختر مصدرًا واحدًا للحقيقة: إما دوال `PostPolicy`، وإما قدرة CASL تُستدعى من تلك الدوال إن احتجت محرّك شروط ضخمًا. استدعاؤها من داخل السياسة يبقي الخدمة على `authorize` ولا ينشر المكتبة في كل متحكم.

## Mental Model

```text
دورة المكتبة: PostPolicy + AuthorizationModule
CASL: محرّك شروط خارجي
        |
 إن لزم: PostPolicy.update تستشير ability
 لا حارس ثانٍ موازٍ
```

## أبسط مثال

```ts
@Policy()
export class PostPolicy {
  constructor(private readonly casl: CaslFactory) {}

  update(user: PublicUser | null, post: { authorId: number }) {
    if (!user) return false;
    return this.casl.for(user).can("update", { ...post, kind: "Post" });
  }
}
```

`CaslFactory` مزوّد تكتبه أنت. هذا جسر محتمل، لا ملف الدورة الافتراضي. الافتراضي دالة `user.id === post.authorId` بلا تبعية.

## شرح المثال

الجسر يبقي `@Can` و `authorize` كما في الصفحة الرسمية، وينقل شرطًا معقدًا إلى CASL. اختبار السياسة ما زال ممكنًا بمزيف `casl`. إن استدعى المتحكم `ability.can` مباشرة، تجاوزت الحزمة وازدوجت الرسائل `403`.

لا تضف هذا البنّاء في مشروع الدورة ما دامت ثلاث دوال تغطي النشر والتعديل والقراءة.

## ماذا يحدث داخليًا؟

CASL يقيّم الشروط على كائن عادي. لا يفتح PostgreSQL ولا يعرف `PrismaService`. تحميل المقال يبقى في الخدمة قبل النداء، وإلا فالحقل `authorId` غير موجود وتفشل القاعدة بصمت أو تمنع الجميع. هذا القيد نفسه الذي منع `@Can` على `update`.

قواعد CASL في الذاكرة لكل طلب. بناء القدرة رخيص عند ثلاث قواعد، ومفيد حين تُولَّد القواعد من صفوف صلاحيات بالمئات بشكل متكرر. عندها المصنع يقرأ تلك الصفوف مرة.

## مثال عملي

مكتبة الدورة: لا حزمة `@casl/ability` في `package.json`. `PostPolicy` دوال صريحة. مشروع شركة فيه مئة شرط `can` على أنواع المستندات: أبقِ CASL هناك، وضع نداءً واحدًا خلف سياسة أو خلف خدمة تفويض حتى لا يتعلم كل مسار API المكتبة. لا تهاجر ذلك المشروع إلى `@Policy` في أسبوع الصيانة نفسه إلا بخطة اختبار لكل فعل.

## أخطاء شائعة

- تثبيت CASL لأن مقالًا أقدم من صفحة التفويض الحالية.
- حارس CASL وحارس `AuthorizationModule` على المسار نفسه.
- شرط CASL على مقال لم يُحمَّل.
- اعتبار CASL جزءًا من `@nestjs/authorization`.

## متى نستخدمه؟

قواعد كثيرة متجانسة، أو codebase قائم على CASL، خلف نقطة ندائية واحدة.

## متى لا نستخدمه؟

لمكتبة الدورة ذات السياسات القليلة. الدالة أوضح من محرّك. ولا كبديل عن المصادقة. CASL لا يثبت كلمة المرور.

## مقارنة

| `@nestjs/authorization` | CASL |
| --- | --- |
| صفحة Security الحالية | مكتبة خارجية |
| `@Policy` و `@Can` | `AbilityBuilder` |
| مسار الدورة | خيار عند كثرة الشروط |
| يتكامل مع حارس المصادقة | تدمجه أنت |

## تمارين

### تمرين سهل

هل CASL يثبت هوية الطلب؟

<details>
<summary>الحل</summary>

لا. هو تقييم أفعال بعد أن تملك مستخدمًا أو ضيفًا. الهوية من فصل المصادقة.

</details>

### تمرين متوسط

لماذا لا نضع `ability.can` في المتحكم بجانب `@Can`؟

<details>
<summary>الحل</summary>

يصبح للمسار قراران وقد يختلفان. مصدر واحد: سياسة الحزمة، وتستدعي CASL من داخلها فقط إن وُجدت حاجة.

</details>

### تمرين متقدم

متى تبقى الدالة `user.id === post.authorId` أفضل من شرط CASL المكافئ؟

<details>
<summary>الحل</summary>

حين تكون القاعدة واحدة وواضحة. قراءة الدالة في اختبار السياسة أرخص من تعلّم شكل الشرط. CASL حين تتكرر العائلة نفسها على أنواع كثيرة.

</details>

## أسئلة مراجعة

- أين تعيش سياسات الدورة؟
- ماذا لا تفعل CASL؟
- كيف تتجنب حارسين؟

## الخلاصة

CASL محرّك قدرات خارجي مفيد عند حجم قواعد كبير أو مشروع قائم. مسار المنشورات يبقى على `@nestjs/authorization` بدوال صريحة.

## ماذا بعد؟

[Guard مقابل Policy](./07-guards-versus-policies.md) أي سؤال يُحسم قبل المعالج وأي سؤال بعد تحميل الصف.

## المصادر الرسمية

- NestJS — Authorization: https://docs.nestjs.com/security/authorization
- NestJS — Guards: https://docs.nestjs.com/guards
- CASL — Ability: https://casl.js.org/v6/en/guide/intro
