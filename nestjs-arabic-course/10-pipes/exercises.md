# تمارين الفصل

أجب عن مسارات الفواتير من غير فتح الدرس.

## سهل

1. ما الواجهة التي يطبّقها أي أنبوب؟
2. أي أنبوب مدمج يملأ استعلامًا غائبًا قبل التحليل؟
3. ماذا يرجع `ParseIntPipe` عند النجاح؟

<details>
<summary>الحل</summary>

1. `PipeTransform` ودالتها `transform`.
2. `DefaultValuePipe`.
3. عددًا صحيحًا، أو يرمي `400` إن فشل.

</details>

## متوسط

صمم توقيع `GET /invoices/:ref` و `GET /invoices`:

- `ref` يجب أن يكون UUID إصدار 4.
- `page` غيابها يساوي `1`، وحضورها عدد صحيح.
- `open` منطق إجباري في الاستعلام.

اذكر الأنابيب بالترتيب على كل معامل.

<details>
<summary>الحل</summary>

- `@Param('ref', new ParseUUIDPipe({ version: '4' }))`.
- `@Query('page', new DefaultValuePipe(1), ParseIntPipe)`.
- `@Query('open', ParseBoolPipe)`.
- لا `ValidationPipe` لهذه القيم الثلاث. هي تحليل قيم لا صنف جسم.

</details>

## متقدم

الدالة:

```ts
update(
  @Body() body: UpdateInvoiceDto,
  @Param('id') id: string,
  @Query('echo') echo: string,
)
```

عليها `@UsePipes(TracePipe)` وعلى التطبيق `ValidationPipe` عام. `id` أيضًا عليه `HoldCodePipe`.

1. من يعمل أولًا على `echo`؟
2. متى يعمل `HoldCodePipe` بالنسبة إلى `TracePipe`؟
3. لماذا لا يرى `try/catch` في أول سطر من `update` فشل `HoldCodePipe`؟
4. جسم زائد فيه `role`. هل يمنعه هذا التمرين؟

<details>
<summary>الحل</summary>

1. `ValidationPipe` العام، لأنه قبل `@UsePipes` على الدالة. داخل كل منهما تُزار المعاملات من الأخير: `echo` ثم `id` ثم `body`.
2. بعد أنابيب المسار والمتحكم والعام، ضمن أنابيب المعاملات، وما زال اتجاه المعاملات من الأخير إلى الأول.
3. الرمي يحدث قبل استدعاء الدالة.
4. لا. `whitelist` غير مفعّل في هذا التمرين. الحذف أو الرفض فصل التحقق.

</details>
