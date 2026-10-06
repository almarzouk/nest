# أخطاء النطاق

## الهدف من هذا الدرس

كتابة فشل قواعد متجر رف كأصناف يرميها الـ Service، وتبقى مفهومة لو استُدعيت من مهمة أو من اختبار بلا HTTP.

## المشكلة التي يحلها

`BooksService.publish` تستورد `ConflictException` من `@nestjs/common`. اختبار الوحدة يحتاج سياق Nest ليفهم الخطأ. مهمة CLI التي تنشر كتابًا مجدولًا تلتقط استثناء HTTP لا معنى له في الطرفية. القاعدة دُفنت داخل البروتوكول.

## الفكرة ببساطة

خطأ النطاق جملة عن الكتب: غير موجود، منشور سلفًا، ISBN مكرر، السعر سالب بعد خصم المجال. الصنف يحمل معرّفات تساعد السجل. لا يحمل `getStatus()`.

## التعريف التقني

صنف يمتد من `Error` حتى تبقى سلسلة الاستدعاء. ميّز الأنواع بـ `instanceof` لا بمقارنة `message`. ضع الأصناف في ملف مثل `books.errors.ts` يستورده الـ Service ولا يستورد هو من `@nestjs/common`. إن كان التطبيق HTTP فقط بوعي، بعض الفرق ترمي `HttpException` من الخدمة. هذا الدرس يختار الفصل لأن رف ستُستدعى لاحقًا من مهام و webhooks. التحقق من شكل DTO يبقى في الـ Pipe؛ لا تعد كتابته كخطأ نطاق.

## Mental Model

```text
BooksService
  يرمي BookNotFound | InvalidShelfPrice | IsbnTaken
المتحكم لا يترجم
الفلتر لاحقًا يترجم
```

## أبسط مثال

```ts
export class InvalidShelfPrice extends Error {
  constructor(readonly priceCents: number) {
    super('Shelf price must be zero or greater');
  }
}

export function assertPrice(priceCents: number) {
  if (priceCents < 0) throw new InvalidShelfPrice(priceCents);
}
```

## شرح المثال

- القاعدة «السعر لا يكون سالبًا» تعيش مع الكتالوج حتى لو تغيّر شكل JSON.
- الحقل `priceCents` يُسجَّل. الرسالة للسجل الداخلي، لا عقد للعميل بعد.
- الدالة نقية: اختبارها يستدعي `assertPrice(-1)` ويتوقع الصنف.

## ماذا يحدث داخليًا؟

1. المتحكم يمرر أرقامًا بعد أن قبل الـ Pipe شكلها.
2. الخدمة تحمّل الكتاب وتطبق القاعدة.
3. `throw` يوقف بقية العملية: لا `update` بعد سعر مرفوض.
4. الاستثناء يصعد ككائن `Error` عادي. Nest لا يعرف اسم `InvalidShelfPrice` حتى يسجّل فلتر لذلك.
5. من غير فلتر، الاستثناء غير المتوقع وسيصبح 500. هذا متوقع إلى أن يكتمل الدرس التالي، وليس سببًا لإرجاع `HttpException` إلى الخدمة.

## مثال عملي

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service.js';
import { BookNotFound, IsbnTaken } from './books.errors.js';

@Injectable()
export class BooksService {
  constructor(private readonly prisma: PrismaService) {}

  async assignIsbn(tenantId: string, bookId: string, isbn: string) {
    const book = await this.prisma.book.findFirst({ where: { id: bookId, tenantId } });
    if (!book) throw new BookNotFound(bookId);
    const owner = await this.prisma.book.findFirst({ where: { tenantId, isbn } });
    if (owner && owner.id !== bookId) throw new IsbnTaken(isbn);
    return this.prisma.book.update({ where: { id: bookId }, data: { isbn } });
  }
}
```

`findFirst` يقيّد المستأجر حتى لا يرمي الخدمة `BookNotFound` لكتاب مكتبة أخرى وكأنه غاب عن العالم. `IsbnTaken` قاعدة كتالوج. قيد القاعدة يظل شبكة أمان تحت السباق، ويُلتقط في طبقة واحدة لا داخل كل سطر.

## أخطاء شائعة

- سلسلة `if (!book) return null` ثم نسيان أن المتحكم يعيد 200.
- وضع رسالة عربية موجهة للواجهة داخل `super()`. الترجمة للعميل قرار الحد.
- صنف واحد `DomainError` بحقل `code: string` يتراكم فيه خمسون رمزًا بلا أنواع، فيتعطل `instanceof`.
- التحقق من `@IsISBN()` داخل الخدمة مرة أخرى بعد الـ Pipe.

## متى نستخدمه؟

كلما منعت قاعدة عمل أثرًا: نشر، سعر، تفرّد ISBN، انتقال حالة. ضع الاختبار على الصنف.

## متى لا نستخدمه؟

لفشل شكل JSON، ولفقدان الاتصال، ولرسالة «غير مصرح» التي يملكها الحارس أصلًا. الحارس قرار أمن على الحد، لا قاعدة سعر.

## مقارنة

| | خطأ نطاق | `ConflictException` داخل الخدمة |
| --- | --- | --- |
| الاختبار | `instanceof` بلا Nest | يحتاج سلوك HTTP أو محاكاة |
| المهمة بلا HTTP | مفهومة | تبدو كاستجابة ويب |
| تغيير الرمز من 409 إلى 422 | يمس الفلتر | يمس كل خدمة رمت الاستثناء |

## تمارين

### تمرين سهل

أين يعيش `BookNotFound`: في `books.service.ts` كـ `throw`، أم في `books.controller.ts` كـ `NotFoundException` مباشرة من غير خدمة؟

<details>
<summary>الحل</summary>

الخدمة ترمي `BookNotFound` بعد ما تملك قاعدة «لا كتاب بهذا المعرّف داخل المستأجر». المتحكم لا يعيد اختراع البحث ليرمي استثناء HTTP.

</details>

### تمرين متوسط

اختبار يستدعي `assignIsbn` برقم موجود لكتاب آخر. ماذا يتوقع أن يُرمى، وماذا لا يتوقع على `response`؟

<details>
<summary>الحل</summary>

يتوقع `IsbnTaken`. لا يوجد `response` في اختبار الخدمة. الرمز 409 سيُثبت لاحقًا على الفلتر.

</details>

### تمرين متقدم

سباق طلبين يعينان نفس ISBN بعد ما اجتاز كلاهما `findFirst`. أين تبقى القاعدة؟

<details>
<summary>الحل</summary>

قيد فريد في Postgres على `(tenantId, isbn)`. الخدمة ترمي `IsbnTaken` للمسار السعيد المقروء. انتهاك القيد يُترجم إلى نفس النوع أو إلى نفس خانة التعارض في الفلتر، حتى لا يعتمد الكتالوج على الترتيب فقط.

</details>

## أسئلة مراجعة

- لماذا لا تستورد `books.errors.ts` من `@nestjs/common`؟
- بماذا تميّز الأنواع في `catch`؟
- ماذا يحدث اليوم لخطأ النطاق إذا لم يوجد فلتر بعد؟
- أي فشل يبقى في الـ Pipe؟

## الخلاصة

خطأ النطاق صنف `Error` بلغة الكتب. الخدمة ترميه وتتوقف عن الأثر. HTTP ليس جزءًا من تعريفه.

## ماذا بعد؟

الأصناف جاهزة ولا يراها العميل. الدرس التالي يترجم كل صنف إلى وضع واستجابة عند الحد فقط.

## المصادر الرسمية

- NestJS — Exception filters: https://docs.nestjs.com/exception-filters
- NestJS — Providers: https://docs.nestjs.com/providers
