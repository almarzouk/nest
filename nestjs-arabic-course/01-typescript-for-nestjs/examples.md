# أمثلة عملية

تجميع لغة الفصل على أصناف تشبه Nest، من غير تشغيل الإطار. الوسوم تُذكر حيث يلزم Metadata، وتسجيل الوحدة ما زال مؤجلًا إلى فصل Modules.

## المثال 1 — حدود النوع والاستنتاج

```ts
interface CreateBook {
  title: string;
  price: number;
}

function draftLabel(input: CreateBook): string {
  const label = `${input.title} (${input.price})`;
  return label;
}
```

`label` يُستنتج `string`. `CreateBook` واجهة ممحوة. هذا العقد بين دوالك، لا فحص JSON.

## المثال 2 — غياب واتحاد

```ts
type Book = { id: string; title: string; price: number };

function find(id: string, rows: Book[]): Book | null {
  return rows.find((row) => row.id === id) ?? null;
}
```

المستدعي يفحص `null` قبل `title`. المصفوفة الفارغة تعيد `null` بسبب `??`.

## المثال 3 — السطر الكامل

```ts
class BooksService {
  async findOne(id: string): Promise<Book | null> {
    return id === '2' ? { id, title: 'Refactoring', price: 40 } : null;
  }
}

class BooksController {
  constructor(private readonly booksService: BooksService) {}

  async detail(id: string): Promise<string> {
    const book = await this.booksService.findOne(id);
    if (!book) {
      return 'missing';
    }
    return book.title;
  }
}

const controller = new BooksController(new BooksService());
```

- `private readonly` يصرح بالحقل ويمنعه من إعادة التعيين في الفحص.
- `await` يفك `Promise`.
- `new` هنا للتعليم. Nest ستمرر النسخة المسجّلة.
- `'missing'` كنص ليس عقد HTTP. لاحقًا ترمي استثناء يحمل `404`.

## المثال 4 — زينة ومخزن وصف

```ts
import 'reflect-metadata';

const PATH = Symbol('path');

function Path(path: string): ClassDecorator {
  return (target) => {
    Reflect.defineMetadata(PATH, path, target);
  };
}

@Path('/books')
class Catalog {}

Reflect.getMetadata(PATH, Catalog);
```

التحميل يخزن `'/books'` على الصنف. لا منفذ مفتوح. `Symbol` مفتاح يصعب أن يصطدم بنص عشوائي. Nest تستخدم مفاتيحها الخاصة بالفكرة نفسها.

## المثال 5 — ما الذي لا يفعله النوع

```ts
async function pretend<T>(raw: string): Promise<T> {
  return JSON.parse(raw) as T;
}
```

`as T` يسكت المترجم ولا يفحص `price`. أبقِ المثال في رأسك كتحذير، لا كنمط. الفحص الحي في فصلي DTO و Validation.

## ماذا بعد؟

حل تمارين الفصل ثم انتقل إلى إنشاء المشروع.
