# تمارين الفصل

عقود نادي القراءة. أجب ثم افتح الحل.

## سهل

1. لماذا لا تُفحص `interface` بـ `ValidationPipe`؟
2. أي حزمة تعرّف `@IsEmail()`؟
3. أي دالة تبني نسخة صنف من كائن عادي؟

<details>
<summary>الحل</summary>

1. الواجهة تُمسح عند الترجمة ولا تترك زينة.
2. `class-validator`.
3. `plainToInstance`.

</details>

## متوسط

صمم ثلاثة أسماء لعملية تسجيل عضو وقراءة عضوه:

- ما يرسله العميل عند الإنشاء، وفيه كلمة مرور.
- ما يُخزَّن، وفيه تجزئة.
- ما يرجع به `GET /members/:id`.

اذكر حقلًا واحدًا يعيش في كل واحد فقط.

<details>
<summary>الحل</summary>

- `CreateMemberDto`: `password` فقط هنا من بين الثلاثة المعروضة للعميل.
- كيان `Member`: `passwordHash` لا يخرج.
- `MemberResponseDto`: `joinedAt` أو `id` يولّدهما الخادم ويظهران في الرد ويغيبان عن الإنشاء.
- لا تضع `passwordHash` على صنف الرد، ولا `password` على الكيان بعد التجزئة.

</details>

## متقدم

المسار `POST /members` يعمل اليوم بـ `CreateMemberDto`. مسار جديد `POST /invites` سيُكتب بمخطط Zod. راجع الاقتراحات:

1. حذف `class-validator` لأن التوثيق ذكر Zod.
2. نسخ قاعدة البريد في الزينة وفي `inviteSchema` على `POST /members`.
3. إبقاء التسجيل على الصنف، وكتابة `inviteSchema` للدعوات فقط، وتأجيل `schema` على `@Body` إلى فصل التحقق.
4. `import type { CreateMemberDto }` في المتحكم.

أي رقم تُبقي، وما عيب كل رقم آخر؟

<details>
<summary>الحل</summary>

تُبقي 3. المصدر واحد لكل عملية، والمخطط يُربَط لاحقًا بالأنبوب الذي يفهمه.

1. `class-validator` غير ملغاة، و `ValidationPipe` ما زال مسار الأصناف.
2. قاعدتان على العملية نفسها تفترقان.
4. الاستيراد النوعّي يمسح الصنف وقت التشغيل فيتعطل الفحص.

</details>
