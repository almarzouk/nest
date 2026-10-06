# أمثلة عملية

يوم على مستودع رف ESM.

## المثال 1 — إنشاء وتشغيل

`npx @nestjs/cli@latest new raf-api` مع ESM فيأتي Vitest و oxlint و Prettier. `npm run start:dev` يراقب ويعيد تشغيل العملية. `nest generate controller projects --dry-run` قبل الكتابة. `nest info` قبل ترقية.

## المثال 2 — مساحة وعامل

`nest generate app worker` يحوّل المجلد ويضع `"builder": "rspack"`. `nest g library membership` يصدّر `@app/membership` من `libs/membership/src/index.ts`. Vitest يكرر المسار في `resolve.alias`. لا `--webpack` في السكربتات.

## المثال 3 — طرفية بلا منفذ

تطوير أسرع على التطبيق القياسي: `npm run start -- -b swc` مع `typeCheck`. استكشاف الخدمة: `repl.ts` و `npm run start -- --entryFile repl` ثم `get(ProjectsService)` مع معرف المنظمة وسيطًا. التصدير الليلي: `createApplicationContext` ثم `app.close()`، أو `CommandFactory.run` لأمر `export-invoices`. Devtools بـ `snapshot: true` و `http` خارج الإنتاج فقط.
