# تمارين الفصل

## سهل

أي مبدأ يخرقه مزيف `find` يرمي بينما Prisma يعيد `null`؟

<details>
<summary>الحل</summary>

Liskov Substitution. البديل لا يقف مكان الأصل أمام المستدعي الذي يفحص `null`.

</details>

## متوسط

التقرير والقناة الإدارية يحقنان `Store` فيها قراءة وكتابة وبريد. ماذا تقسم، وكيف تبقى نسخة Prisma واحدة؟

<details>
<summary>الحل</summary>

واجهتا `BooksRead` و `BooksWrite` ورمز لكل منهما. `useExisting` نحو صنف واحد. التقرير لا يحقن الكتابة ولا البريد. البريد عقد ثالث إن كان مستهلكه مختلفًا.

</details>

## متقدم

`CheckoutService` فيها `switch` لثلاث قنوات وتستورد `PrismaClient`. طبّق مبدأين معًا بما يغيّر الملفات.

<details>
<summary>الحل</summary>

Open/Closed: عقد `Payment` وقائمة مزودات. Dependency Inversion: `OrdersPort` بجانب السياسة ومحول Prisma في `useClass` أو `useExisting`. المتحكم يبقى على `place` بلا SQL ولا فروع قنوات.

</details>
