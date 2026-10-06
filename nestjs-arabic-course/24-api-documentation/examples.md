# أمثلة عملية

وثيقة واحدة لمقالات المكتبة. الاستيراد من `@nestjs/swagger`، ومسارات ESM تنتهي بـ `.js` حين تستورد ملفاتك.

## مثال 1 — الإقلاع

`DocumentBuilder` يضع العنوان `Library posts` والإصدار، ثم `addCookieAuth("__Host-sid")` و `addBearerAuth()`. `documentFactory` يستدعي `createDocument`. `setup("api", app, documentFactory)` قبل `listen`. على Fastify تُثبَّت `@fastify/static`.

## مثال 2 — مقال

`CreatePostDto` فيه `title` فقط كحقل إلزامي، مع محقق و `@ApiProperty` أو مع إضافة CLI. `POST` عليه `@ApiCreatedResponse({ type: PostResponseDto })` و `@ApiBearerAuth` أو كعكة، و `@ApiUnauthorizedResponse`. `GET /posts/:id` يعيد النوع نفسه مع `@ApiNotFoundResponse`. `passwordHash` ليس في `PostResponseDto`.

## مثال 3 — صفحة

`GET /posts` يستخدم مزخرف `ApiPaginatedResponse(PostResponseDto)` مبنيًا من `allOf` و `getSchemaPath` و `title`. الغلاف حقوله `items` و `page` و `pageSize` و `total`. `@ApiExtraModels` يسجّل الغلاف. `ListPostsQueryDto` يوثّق السقف `50`.

## ماذا تتأكد منه

- `/api` يعرض `POST /posts` بقفل و `GET /posts` العام بلا قفل إن كانت القراءة `@Public`.
- `api-json` يخلو من `authorId` في جسم الإنشاء ومن `passwordHash` في الاستجابة.
- `PartialType` مستورد من `@nestjs/swagger` بعد تفعيل الإضافة.
- أمثلة الواجهة سلاسل وهمية.
