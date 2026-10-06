# التمهيد للفصل التالي

الإعداد صار طبقة إقلاع: ملف محلي خارج Git، `ConfigModule.forRoot`، قراءة عبر `ConfigService`، ومخطط يرفض النقص قبل `listen`.

أول قيمة ستخرج من هذه الطبقة إلى بقية الدورة هي `DATABASE_URL`. الفصل التالي لا يبدأ باختيار مكتبة. يبدأ بمعنى الاتصال والجدول والترحيل، ثم PostgreSQL، ثم ما يفعله ORM، ثم Prisma كمسار تطبيقي: `PrismaService` يقرأ العنوان من `ConfigService`، يتصل في `onModuleInit`، ويقطع في `onModuleDestroy`.

TypeORM الذي ظهر في معاينة `forRootAsync` يبقى بديلًا موثّقًا. Drizzle عبر `@nestjs/drizzle` بديل موثّق آخر. لن نفتح ثلاث أدوات في المكتبة نفسها.

افتح [قاعدة البيانات](../18-database/README.md) وأنت قادر على تفسير لماذا عنوان القاعدة لا يُكتب داخل `BooksService`.
