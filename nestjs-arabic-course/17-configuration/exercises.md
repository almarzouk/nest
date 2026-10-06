# تمارين الفصل

أغلق الدروس ثم أجب. المكتبة هي السيناريو. لا ترفع ملفًا فيه سر حقيقي.

## سهل

1. صنّف: `PORT`، الحد الأقصى لحروف عنوان الكتاب، `DATABASE_URL`، شعار نصي ثابت في الواجهة.
2. ماذا يحتوي `.env.example` لمفتاح `DATABASE_URL`؟
3. أين يُستدعى `ConfigModule.forRoot`؟

<details>
<summary>الحل</summary>

1. `PORT` و `DATABASE_URL` إعداد. طول العنوان قاعدة مجال. الشعار الثابت كود أو أصل واجهة، لا سر.
2. الاسم وقيمة فارغة أو وصفًا. العنوان الحقيقي في `.env` المحلي أو في المنصة.
3. مرة في `AppModule`. مع `isGlobal: true` لا تعاده في كل ميزة.

</details>

## متوسط

مخطط Zod الحالي:

```ts
z.object({
  PORT: z.string().default("3000"),
  DATABASE_URL: z.string().optional(),
});
```

1. لماذا قد يقلع التطبيق ويتصل لاحقًا بعنوان `undefined`؟
2. أعد صياغة الحقلين حتى يُجبر المنفذ ويُرفض عنوان فارغ.
3. أين تقرأ `PORT` لـ `listen` حتى ترى النوع بعد الإكراه؟

<details>
<summary>الحل</summary>

1. `optional()` يجعل الغياب مقبولًا، و `z.string()` يترك المنفذ نصًا. الاتصال يتأجل إلى أول استعلام.
2. `PORT: z.coerce.number().int().min(1).max(65535).default(3000)` و `DATABASE_URL: z.string().min(1)`.
3. `app.get(ConfigService)` بعد `create`، أو خدمة تُحقن وتستدعي `get("PORT", { infer: true })` على نوع `z.infer`.

</details>

## متقدم

`app.module.ts` فيه:

```ts
const databaseUrl = process.env.DATABASE_URL;

TypeOrmModule.forRoot({
  type: "postgres",
  url: databaseUrl,
  synchronize: true,
});
```

`.env` صحيح، ومع ذلك `url` فارغ في التطوير، والمثال يفتح `synchronize: true`.

1. لماذا العنوان فارغ رغم الملف؟
2. ما شكل `forRootAsync` الذي يحقن `ConfigService`؟
3. لماذا تُطفأ `synchronize` قبل أي فصل ترحيل؟
4. ما علاقة ذلك بمسار Prisma في الفصل التالي؟

<details>
<summary>الحل</summary>

1. السطر أعلى الملف يُقيَّم عند الاستيراد، غالبًا قبل دمج dotenv داخل `forRoot`.
2. `TypeOrmModule.forRootAsync({ inject: [ConfigService], useFactory: (config) => ({ type: "postgres", url: config.getOrThrow("DATABASE_URL"), autoLoadEntities: true, synchronize: false }) })`.
3. توثيق TypeORM يحذّر أن `synchronize: true` قد يمسح بيانات الإنتاج. تغيير المخطط يتم بترحيل.
4. Prisma مسار الدورة التطبيقي. سيقرأ العنوان من `ConfigService` داخل `PrismaService` بعد التحقق. `forRootAsync` هنا توقيت التركيب، لا إلزام باختيار TypeORM.

</details>
