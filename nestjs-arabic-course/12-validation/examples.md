# أمثلة عملية

ثلاثة إعدادات لصالة حجوزات. كل مثال خيار أو أنبوب واحد لا يجمعها كلها في كتلة مبهمة.

## مثال 1 — تبييض ورفض على صنف

```ts
import { IsInt, IsString, Min } from 'class-validator';

export class ReserveSeatDto {
  @IsString()
  showId: string;

  @IsInt()
  @Min(1)
  seat: number;
}
```

```ts
new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
  errorFormat: 'grouped',
})
```

`{ "showId": "s1", "seat": 4, "role": "admin" }` يرجع `400` بسبب `role`. `{ "showId": "s1", "seat": 4 }` يصل نسخة `ReserveSeatDto` لأن `transform: true`.

## مثال 2 — عنوان متداخل

```ts
import { Type } from 'class-transformer';
import { IsString, MinLength, ValidateNested } from 'class-validator';

export class BillingDto {
  @IsString()
  @MinLength(2)
  city: string;
}

export class GiftSeatDto {
  @ValidateNested()
  @Type(() => BillingDto)
  billing: BillingDto;
}
```

`billing.city` من حرف واحد يفشل. حذف `@Type` يوقف فحص المدينة. `each` غير لازم لأن الحقل كائن واحد.

## مثال 3 — استعلام بمخطط بجانب الصنف

```ts
import { z } from 'zod';

export const seatQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
});
```

```ts
app.useGlobalPipes(
  new ValidationPipe({ whitelist: true, transform: true }),
  new StandardSchemaValidationPipe(),
);
```

```ts
@Get()
list(@Query({ schema: seatQuerySchema }) query: { page: number }) {
  return query;
}
```

`POST` الذي ما زال على `ReserveSeatDto` يفحصه الأنبوب الأول. `GET` يفحصه الثاني. `page` الغائب يصبح `1`. لا `@Min` على `page` في هذا المسار.

## ماذا تتأكد منه

- كل حقل مقبول على الصنف عليه زينة قبل `whitelist`.
- `@ValidateNested` و `@Type` معًا على المتداخل.
- مسار المخطط يعلن `{ schema }` ولا يعيد الزينة نفسها.
- `disableErrorMessages` لا يُشغَّل في أمثلة التطوير هذه حتى تبقى الرسائل ظاهرة في الاختبار.
