# المقدمة: قبل NestJS

**الحالة:** مكتمل

هذا الفصل لا يحتوي كود NestJS مطلوبًا للحفظ. هدفه أن ترى المشكلة التي بُني الإطار لحلها: تطبيق خادم يكبر، فيصبح ترتيب المسؤوليات أهم من القدرة على إرجاع `Hello World`.

## ماذا ستقدر أن تفعل بعد هذا الفصل؟

- تصف رحلة طلب من Client إلى Server وبالعكس.
- تقرأ Request و Response: الطريقة، والمسار، والرؤوس، والجسم، ورمز الحالة.
- تميّز REST كأسلوب تصميم، و JSON كصيغة تمثيل، و API كعقد بين طرفين.
- تفسر لماذا Node.js مناسب كأساس لخادم شبكي، وما حدوده التي ستظهر لاحقًا في الأداء.
- تقرأ أصغر تطبيق Express وتتتبع Middleware حتى الـ Handler.
- تسمي أسباب فوضى مشروع Express كبير، وتربط كل سبب بقرار معماري في NestJS.
- ترسم خريطة Nest: Module و Controller و Provider، ومسار الطلب الرسمي من التوثيق، مع علمك أن كل صندوق سيُشرح وحده.

## لمن يُكتب الفصل؟

تفترض الدروس أنك تقرأ JavaScript و TypeScript، وأنك أرسلت طلب HTTP من قبل. الشرح يعيد بناء النموذج من جهة الخادم، لأن NestJS تعيش هناك.

## ترتيب القراءة

1. [ما هو Backend](./01-what-is-backend.md)
2. [Client و Server](./02-client-and-server.md)
3. [Request و Response](./03-request-and-response.md)
4. [HTTP](./04-http.md)
5. [REST](./05-rest.md)
6. [JSON](./06-json.md)
7. [API و REST API](./07-api-and-rest-api.md)
8. [Node.js](./08-nodejs.md)
9. [Library مقابل Framework](./09-library-vs-framework.md)
10. [ما هو Express](./10-what-is-express.md)
11. [كيف يعمل Express](./11-how-express-works.md)
12. [مشاكل المشاريع الكبيرة في Express](./12-large-express-projects.md)
13. [لماذا أُنشئت NestJS](./13-why-nestjs-exists.md)
14. [NestJS فوق Express و Fastify](./14-nestjs-on-express-and-fastify.md)
15. [لماذا تشبه NestJS إطار Angular](./15-why-nestjs-resembles-angular.md)
16. [Architecture الخاصة بـ NestJS](./16-nestjs-architecture-overview.md)
17. [أمثلة عملية](./examples.md)
18. [تمارين الفصل](./exercises.md)
19. [التمهيد للفصل التالي](./next-step.md)

كل درس فيه تمارينه. ملف `exercises.md` يجمع مسائل تربط الدروس ببعضها.

## ما لن نفعله هنا

- لن نثبّت Nest CLI.
- لن نشرح `@Module` أو `@Injectable` كأدوات تستخدمها. ستظهر أسماؤها في الخريطة الأخيرة مع تنبيه أنها ستُشرح لاحقًا.
- لن نكتب مشروع Users كاملًا. الأمثلة تبقى صغيرة حتى تُبنى أجزاؤها في فصولها.

## موقعه في المسار

- التالي: [01-typescript-for-nestjs](../01-typescript-for-nestjs/README.md)

اقرأ [خريطة الدورة](../ROADMAP.md) لترى لماذا يأتي TypeScript قبل إنشاء المشروع.
