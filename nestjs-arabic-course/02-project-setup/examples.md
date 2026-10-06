# أمثلة عملية

## إنشاء مشروع التجربة

من مجلد خارج مواد الشرح:

```bash
node -v
npm i -g @nestjs/cli
nest new bookstore
cd bookstore
npm run start:dev
```

اختر ESM، واترك `strict`، وتجاوز Observe الآن إن أردت ملف إقلاع مطابق للفصول القادمة. افتح `http://localhost:3000/` وتوقع `Hello World!`.

## تتبع النص

1. `AppController.getHello` ينادي الخدمة.
2. `AppService.getHello` ترجع النص.
3. غيّر النص، احفظ، أعد الطلب مع `start:dev`.

## بناء

```bash
npm run build
npm run start:prod
```

أوقف `start:dev` أولًا حتى يتحرر المنفذ.

## ماذا بعد؟

تمارين الفصل ثم الإقلاع بالتفصيل.
