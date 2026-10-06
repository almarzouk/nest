# تمارين الفصل

صالة حجوزات. أجب ثم افتح الحل.

## سهل

1. أي خيار يحذف مفتاحًا بلا زينة من صنف؟
2. ما القيمتان الموثّقتان لـ `errorFormat`؟
3. أي زينة من `class-transformer` تبني الكائن المتداخل؟

<details>
<summary>الحل</summary>

1. `whitelist: true`.
2. `'list'` و `'grouped'`.
3. `@Type(() => ...)`، ومعها `@ValidateNested()` حتى يُفحص الداخل.

</details>

## متوسط

اضبط أنبوب أصناف واحدًا يحقق:

- رفض `role` الزائد بـ `400` لا حذفه الصامت.
- المعالج يستلم `instanceof ReserveSeatDto`.
- رسائل التطوير مجمّعة حسب الحقل.

لا تخفِ الرسائل في هذا التمرين.

<details>
<summary>الحل</summary>

```ts
new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  errorFormat: 'grouped',
})
```

`forbidNonWhitelisted` مع `whitelist` للرفض. `transform` للنسخة. `grouped` لتجميع الرسائل.

</details>

## متقدم

التطبيق فيه `CreateMemberDto` و `reserveSchema`. راجع الخطة:

1. حذف `class-validator`.
2. تسجيل الأنبوبين عامين.
3. `@Body({ schema: reserveSchema })` للحجز، و `@Body() dto: CreateMemberDto` للعضو.
4. `forbidNonWhitelisted` لرفض مفتاح زائد داخل Zod.
5. `@ValidateNested()` من غير `@Type` على `billing`.

أي البنود تُبقي، وما أثر كل بند مرفوض؟

<details>
<summary>الحل</summary>

تُبقي 2 و 3.

1. الأصناف المزيّنة ما زالت مسارًا مدعومًا.
4. رفض مفتاح مخطط Zod يُكتب `z.strictObject` لا خيار أنبوب الأصناف.
5. من غير `@Type` لا تُبنى `BillingDto` وقد لا تُفحص `city`.

</details>
