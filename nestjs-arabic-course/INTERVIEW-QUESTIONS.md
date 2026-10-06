# أسئلة مقابلات

الجواب المتوقع يشرح السبب. الأسئلة 1–25 من المقدمة، و26–42 من بقية المسار.

## المقدمة — HTTP و REST

1. لماذا نعيد التحقق على الخادم إذا كانت الواجهة تتحقق؟
2. ما الذي يعرّف Client في محادثة واحدة؟ هل تستطيع عملية Nest أن تكون Client؟
3. ما أجزاء Request، وما الذي يضيفه Status ولا يغني عنه حقل `error` داخل JSON؟
4. ماذا يعني أن HTTP بروتوكول Stateless، وأين تعيش حالة المستخدم إذًا؟
5. لماذا يُوصف `GET` بأنه safe، وماذا ينكسر إذا نشرنا كتابًا عبر `GET`؟
6. ما الفرق العملي بين `PUT` و `PATCH` و `POST` عند التكرار؟
7. متى تستخدم `401` ومتى تستخدم `403`؟
8. لماذا لا يكفي إرجاع JSON لنسمي الواجهة REST؟
9. ما الفرق بين المورد وتمثيله؟
10. لماذا نوع TypeScript على جسم الطلب لا يحمي الإنتاج؟

## المقدمة — Node و Express و Nest

11. لماذا لا يسرّع `async/await` حلقة حساب طويلة على خيط JavaScript؟
12. ما الفرق بين مكتبة تناديها وإطار يناديك؟ أين يقع Express، وأين تقع Nest؟
13. ماذا يفعل `next()` في Express، وماذا يحدث إذا لم تُستدعَ ولم يُنهَ الرد؟
14. لماذا `req` في Express ما زال كائن Node؟
15. ما أعراض مشروع Express صارت المعمارية مشكلته لا التوجيه؟
16. ما المشكلة التي تسميها صفحة Nest الأولى قبل Decorators؟
17. ما الحزم الدنيا لمشروع Nest من غير CLI؟
18. ماذا يستقبل `NestFactory.create`، وما المنصة الافتراضية تحته؟
19. ما وظيفة HTTP Adapter، ولماذا بقي Express الافتراضي رغم حديث التوثيق عن سرعة Fastify؟
20. ماذا يشترك مع Angular، وماذا لا ينتقل من الواجهة إلى الخادم؟
21. ما الفرق بين Provider و Service في جملة واحدة؟
22. لماذا تسجيل Controller في `@Module` شرط، ووجود الملف لا يكفي؟
23. رتّب مسار الطلب الناجح: Pipe، Guard، Middleware، Interceptor قبل المتحكم، Controller.
24. من أي مستوى يبدأ اختيار Exception Filter، وماذا يختلف استثناء Middleware؟
25. لماذا تُرجع دالة المتحكم قيمة في الأسلوب القياسي بدل استدعاء `res.send`؟ ما رمز `POST` الافتراضي؟

## من المسار الكامل

26. لماذا تُغلَّف المزوّدات داخل الوحدة، وماذا يغيّر `exports`؟
27. ما ترتيب Request Lifecycle حتى المتحكم، وكيف تُختار الفلاتر؟
28. لماذا Interface لا تكفي لـ `ValidationPipe`؟ وماذا يضيف `StandardSchemaValidationPipe`؟
29. ما الفرق بين Middleware و Guard و Pipe و Interceptor و Exception Filter؟
30. لماذا `@Res()` بدون `passthrough` يغيّر من يكتب الرد؟
31. ما النطاق الافتراضي للحقن، ولماذا `REQUEST` مكلف؟
32. متى يكون `forwardRef` عرضًا لا علاجًا؟
33. ماذا يُخزَّن بدل كلمة المرور، وأين يعيش سر التوقيع؟
34. ما الفرق بين 401 و 403؟
35. لماذا قائمة فارغة 200 لا 404؟
36. كيف تمنع أثر الويب هوك المكرر؟
37. لماذا لا تؤكد صفحة نجاح الدفع وحدها الاشتراك؟
38. ما فرق الجلسة عن JWT من جهة الإبطال؟
39. لماذا نمرر `bookId` إلى BullMQ لا كائن Prisma؟
40. متى تبقى على مونوليث الوحدات بدل Microservices؟
41. ما فرق liveness عن readiness؟
42. ماذا تشغّل في الإنتاج: `start:dev` أم ناتج `dist`؟

## أجوبة مختصرة للمراجعة

<details>
<summary>23 — الترتيب</summary>

Middleware، ثم Guard، ثم Interceptor قبل المتحكم، ثم Pipe، ثم Controller.

</details>

<details>
<summary>24 — الفلاتر</summary>

من Route ثم Controller ثم Global. استثناء Middleware تراه الفلاتر العامة فقط.

</details>

<details>
<summary>25 — الرد القياسي</summary>

Nest تكتب الحالة والجسم عندما تُرجع قيمة. الافتراضي `200`، و `POST` يستخدم `201` إلا إذا غيّرته بـ `@HttpCode`. الأسلوب الآخر حقن `@Res()` وهو library-specific ويُشرح في فصل Controllers.

</details>

<details>
<summary>26 — exports</summary>

المزوّد خاص بوحدته. `exports` يفتحه لمن يستورد الوحدة، مع مشاركة التسجيل نفسه.

</details>

<details>
<summary>27 — lifecycle</summary>

Middleware ثم Guard ثم Interceptor قبل المتحكم ثم Pipe ثم Controller. الفلاتر من المسار إلى المتحكم إلى العام. استثناء Middleware تراه الفلاتر العامة فقط.

</details>

<details>
<summary>28 — DTO</summary>

Interface تُمحى قبل التشغيل فلا يوجد ما يفحصه `ValidationPipe`. الصنف يبقى. `StandardSchemaValidationPipe` يفحص مخططات مثل Zod، ويمكن أن يعمل مع الأنبوب الأول.

</details>

<details>
<summary>31 — REQUEST</summary>

الافتراضي singleton. `REQUEST` نسخة لكل طلب، وأي singleton يحقنه يصير على النطاق نفسه، فيرتفع عدد الإنشاءات.

</details>

<details>
<summary>33 — كلمة المرور</summary>

يُخزَّن ناتج تجزئة بطيئة مخصصة لكلمات المرور. سر توقيع الرمز يعيش في الإعداد على الخادم، لا في المستودع ولا في الرد.

</details>

<details>
<summary>36 — تكرار الويب هوك</summary>

مفتاح تكرار من معرّف الحدث، مع حفظ الأثر مرة واحدة. التحقق من التوقيع على الجسم الخام قبل قراءة JSON.

</details>

<details>
<summary>42 — الإنتاج</summary>

`npm run build` ثم تشغيل `dist`. `start:dev` يراقب الملفات ولا يناسب صورة الإنتاج.

</details>
