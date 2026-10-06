# أمثلة عملية

ثلاثة مسارات على موارد مختلفة. كل مثال يوضح قرارًا واحدًا من هذا الفصل: إرجاع القيمة، قراءة المدخلات، ومن يكتب الاستجابة. الاستيراد النسبي ينتهي بـ `.js`.

## مثال 1 — JSON قياسي لطلبات النسخ

`POST` يعيد الكائن برمز 201. `GET reprint` ثابت قبل `:id` حتى لا تُحجب الكلمة على Express الافتراضي.

```ts
import { Controller, Get, Param, Post } from '@nestjs/common';
import { CopiesService } from './copies.service.js';

@Controller('copies')
export class CopiesController {
  constructor(private readonly copies: CopiesService) {}

  @Get('reprint')
  reprintQueue() {
    return this.copies.reprintQueue();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.copies.findOne(id);
  }

  @Post()
  create() {
    return this.copies.createDraft();
  }
}
```

`GET /copies/reprint` قائمة. `GET /copies/c-18` نسخة واحدة و `id` نص. `POST /copies` يعيد مسودة JSON برمز 201 لأن الطريقة POST ومن غير `@HttpCode`.

## مثال 2 — رف في المسار وصفحة في الاستعلام

الهوية في المسار، والترقيم في `?`. `page` نص حتى يأتي Pipe.

```ts
import { Controller, Get, Param, Query } from '@nestjs/common';
import { AislesService } from './aisles.service.js';

@Controller('aisles')
export class AislesController {
  constructor(private readonly aisles: AislesService) {}

  @Get(':code/bins')
  bins(@Param('code') code: string, @Query('page') page = '1') {
    return this.aisles.bins(code, page);
  }
}
```

`GET /aisles/north/bins?page=3` يمرر `code` بالقيمة `north` و `page` بالقيمة `"3"`. طلب بلا `page` يستخدم `'1'` لأن الغياب `undefined`.

## مثال 3 — CSV خام، cookie مع JSON، وتحويل دليل

كل مسار يختار من يكتب الاستجابة.

```ts
import {
  Controller,
  Get,
  Header,
  HttpCode,
  HttpStatus,
  Post,
  Query,
  Redirect,
  Res,
} from '@nestjs/common';
import type { Response } from 'express';
import { DeskService } from './desk.service.js';

@Controller('desk')
export class DeskController {
  constructor(private readonly desk: DeskService) {}

  @Get('queue.csv')
  queue(@Res() res: Response) {
    res.type('text/csv');
    res.send(this.desk.queueCsv());
  }

  @Post('ack')
  @HttpCode(HttpStatus.OK)
  @Header('Cache-Control', 'no-store')
  ack(@Res({ passthrough: true }) res: Response) {
    res.cookie('desk', 'acked', { httpOnly: true });
    return { acked: true };
  }

  @Get('guide')
  @Redirect('https://docs.example.com/desk', 302)
  guide(@Query('year') year?: string) {
    if (year === '2026') {
      return { url: 'https://docs.example.com/desk/2026' };
    }
  }
}
```

`queue.csv` وضع مكتبة: `res.send` ينهي الرد، والقيمة المعادة غير مستخدمة. `ack` يضبط cookie ثم تعيد الدالة كائنًا، فترسله Nest JSON برمز 200 بسبب `@HttpCode` على POST. `guide` يحوّل إلى الدليل، و `?year=2026` يستبدل العنوان فقط فيبقى الرمز 302.

## ماذا تتأكد منه

- كل Controller مذكور في `controllers` داخل Module تصل إلى الجذر.
- المسار الثابت مكتوب قبل `:id` ما دامت الاستراتيجية `declaration`.
- `@Res()` بلا `passthrough` يرافقه `res.send` أو ما يعادله.
- جسم JSON العادي يُعاد كقيمة، والتحقق من الحقول يأتي مع Pipes لاحقًا.
