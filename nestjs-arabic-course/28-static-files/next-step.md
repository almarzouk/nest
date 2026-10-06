# التمهيد للفصل التالي

الملفات العامة لها باب، والـ API له باب. الفصل `29-graphql` يغيّر شكل الباب الثاني: العميل يطلب شجرة حقول بدل ما يستدعي مسارًا لكل شكل شاشة.

## ماذا يبقى من هذا الفصل

- الأصل العام في `public` أو `client/dist` عبر `ServeStaticModule`.
- `serveRoot` أو `exclude` حتى لا يبتلع البدل مسارات JSON.
- على Fastify الاستثناء خيار Express. القيد هناك `renderPath`.
- غلاف المجال يبقى `StreamableFile` خارج `rootPath`.
- `GET /api/missing` يجب ألا يعيد `index.html`.

## ماذا ستقرأ هناك

GraphQL لغة استعلام ومخطط. Nest تولّد المخطط من أصناف TypeScript في الأسلوب code-first مع Apollo، وهذا مسار الدورة. Schema-first يُذكر حتى تعرف الفرق. المحللات تقابل المتحكمات، والحجج تقابل المعاملات، والـ context يحمل الطلب. DataLoader يُلمح هنا ويُعمَّق في فصل لاحق. الاشتراكات تستخدم `graphql-ws`، وواجهة التجربة GraphiQL لا Playground القديم.

لا تفتح `@nestjs/graphql` قبل درس «ما هو GraphQL». المخطط مفهوم مستقل عن الزينة.

افتح [`29-graphql`](../29-graphql/README.md) بعد أن ينجح فحص الواجهة وفحص مسار API الغائب على مشروعك.
