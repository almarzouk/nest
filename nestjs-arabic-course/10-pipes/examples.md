# أمثلة عملية

ثلاثة مسارات فواتير. كل مثال مستوى ربط مختلف. الاستيراد المحلي بلاحقة `.js`.

## مثال 1 — صفحة بقيمة افتراضية

`GET /invoices?page=2` عدد. الغياب يعني `1`. النص `abc` لا يدخل الخدمة.

```ts
import { Controller, Get, ParseIntPipe, Query, DefaultValuePipe } from '@nestjs/common';

@Controller('invoices')
export class InvoicesController {
  @Get()
  list(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  ): { page: number } {
    return { page };
  }
}
```

`DefaultValuePipe` يسار `ParseIntPipe` حتى تُملأ القيمة قبل التحليل. لا أنبوب مخصص.

## مثال 2 — قيد حجز بصنف نطاق

```ts
import { BadRequestException, Injectable, type PipeTransform } from '@nestjs/common';

@Injectable()
export class HoldCodePipe implements PipeTransform<string, string> {
  transform(value: string): string {
    const code = value.trim().toUpperCase();
    if (!/^HLD-\d{3,}$/.test(code)) {
      throw new BadRequestException('hold must look like HLD-240');
    }
    return code;
  }
}
```

```ts
@Get('holds/:hold')
findHold(@Param('hold', HoldCodePipe) hold: string): { hold: string } {
  return { hold };
}
```

`hld-15` يُرفض لقصر الأرقام. ` hld-240 ` يصبح `HLD-240`. الغياب في القاعدة `404` من الخدمة لاحقًا.

## مثال 3 — جسم مع أنبوب عام ومعامل مسار

```ts
import { IsInt, IsString, Min } from 'class-validator';

export class CreateInvoiceDto {
  @IsString()
  customer: string;

  @IsInt()
  @Min(1)
  cents: number;
}
```

```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module.js';

const app = await NestFactory.create(AppModule);
app.useGlobalPipes(new ValidationPipe());
await app.listen(3000);
```

```ts
@Post(':desk')
create(
  @Param('desk', ParseIntPipe) desk: number,
  @Body() dto: CreateInvoiceDto,
): CreateInvoiceDto {
  void desk;
  return dto;
}
```

`ParseIntPipe` على `desk` مستقل عن فحص الجسم. جسم بلا `customer` يرجع `400` من `ValidationPipe`. `desk=north` يرجع `400` من أنبوب التحليل. الحقول الزائدة ما زالت تمر لأن `whitelist` ليس في هذا المثال.

ترتيب الزيارة لمعاملَي `create`: أنابيب المستوى الأعلى ترى `dto` ثم `desk` لأن الأخير في التوقيع يُعالَج أولًا داخل ذلك المستوى، ثم أنابيب المعاملات بالاتجاه نفسه. `ParseIntPipe` مربوط بـ `desk` فقط.

## ماذا تتأكد منه

- التحليل المدمج قبل أي صنف محلي يحمل الاسم نفسه.
- الافتراضي مكتوب قبل `Parse*`.
- DTO صنف مُستورَد استيراد قيمة لا `import type`.
- فشل الأنبوب لا يُلتقط بـ `try/catch` داخل الدالة.
