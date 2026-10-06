# تمارين الفصل

## تمرين سهل — سمِّ الخانة

لكل حالة، اكتب: تحقق، غياب، تعارض، أو غير متوقع.

1. `title` ليس نصًا.
2. `DELETE /books/nope` لا يجد صفًا داخل المستأجر.
3. نشر كتاب حالته `published`.
4. انتهت مهلة الاتصال بـ Postgres.

<details>
<summary>الحل</summary>

1 تحقق (Pipe). 2 غياب (`BookNotFound` → 404). 3 تعارض حالة. 4 غير متوقع → 500 في السجل فقط.

</details>

## تمرين متوسط — راجع الالتقاط

```ts
try {
  return await this.books.publish(id);
} catch (error) {
  if (error instanceof HttpException) throw new InternalServerErrorException();
  throw error;
}
```

ماذا يفعل هذا المقطع بفشل الحارس الذي وصل كـ `HttpException`، وما التعديل الذي يوافق الاستراتيجية؟

<details>
<summary>الحل</summary>

يحوّل 401 أو 403 أو 404 الآتية كـ `HttpException` إلى 500. هذا ممنوع. احذف `try/catch` كله ودع `publish` يرمي صنف مجال، والفلتر يترجم، و `HttpException` تمر بوضعها.

</details>

## تمرين متقدم — صف جديد

رف تريد منع سحب كتاب ما زال في طلب شراء مفتوح. صمّم الصنف والرمز ومكان الاختبار من غير كتابة فلتر ثانٍ.

<details>
<summary>الحل</summary>

`BookInOpenOrder extends Error` يرميه `BooksService.withdraw`. سطر في جدول الفلتر الحالي إلى 409. اختبار وحدة يتوقع الصنف. اختبار e2e يتوقع 409 ورسالة بلا تفاصيل الطلب الداخلية. لا متحكم فيه `catch`. إن نسي أحد السطر، ظهور الصنف في سجل 500 يفشل مراجعة المراقبة إلى أن يُضاف.

</details>
