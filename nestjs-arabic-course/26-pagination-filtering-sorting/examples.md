# أمثلة عملية

قائمتان على المكتبة. الاستيراد ينتهي بـ `.js`. الفهارس بهجرة Prisma.

## مثال 1 — لوحة المقالات

`ListPostsQueryDto` يوسّع `PageQueryDto` ويضيف `sort` من `POST_SORTS` و `order`. الزائر نطاقه `{ published: true }`. `orderByPosts` ينهي فرز العنوان بـ `id`. `toSkipTake` و `pageOf` و `toPostResponse`. السقف `50`. الصفحة البعيدة الفارغة `200`. الفهرس `[published, id]` و `[authorId, id]`.

## مثال 2 — خيط تعليقات

`CommentCursorQueryDto` فيه `cursor` و `limit` بلا `page`. `findMany` على `postId` من المسار، `orderBy` على `id`، `cursor` مع `skip: 1`، و `take: limit + 1`. الناتج `{ items, nextCursor }`. الفهرس `[postId, id]`. لا `count` في هذا المسار.

## مثال 3 — ما يُرفض

`sort=passwordHash` يعطي `400`. `published=false` للزائر لا يكشف المسودات. `pageSize=51` يُرفض على اللوحة و `limit=51` يُرفض على الخيط. جسم العنصر بلا `passwordHash`.

## ماذا تتأكد منه

- المقالات والتعليقات لا تشتركان في صنف واحد يجمع `page` و `cursor`.
- `count` و `findMany` يستقبلان `where` نفسه على اللوحة.
- `EXPLAIN` لمسار الزائر يستخدم فهرس `published, id` على بيانات بحجم واقعي.
- الهجرة هي طريق الفهرس، بلا `synchronize: true`.
