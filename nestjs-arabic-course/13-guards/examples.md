# أمثلة عملية

بوابتان على شباك التذاكر، ثم مسار صحة يبقى خارجهما. الترويسة التعليمية `x-desk-name` ليست رمز إنتاج.

## مثال 1 — هوية تلصق اسم الشباك

```ts
import {
  type CanActivate,
  type ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';

type DeskRequest = {
  headers: Record<string, string | undefined>;
  user?: { name: string };
};

@Injectable()
export class IdentifyDeskGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<DeskRequest>();
    const name = request.headers['x-desk-name'];
    if (!name) {
      throw new UnauthorizedException();
    }
    request.user = { name };
    return true;
  }
}
```

غياب الترويسة `401`. وجودها أي نص غير فارغ `true` مع `request.user`.

## مثال 2 — منع الاسترداد لغير `lead`

```ts
@Injectable()
export class RefundGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<DeskRequest>();
    return request.user?.name === 'lead';
  }
}
```

`false` يصبح `403`. الحارس لا يرمي `401` لأن الغياب عولج قبله. إن استُدعي وحده بلا هوية، `user` فارغ والناتج `403` أيضًا، لذلك الترتيب في المثال 3 يضع الهوية أولًا.

## مثال 3 — الربط على المتحكم لا على التطبيق

```ts
@Controller('tickets')
@UseGuards(IdentifyDeskGuard)
export class TicketsController {
  @Get()
  list(): { ok: true } {
    return { ok: true };
  }

  @Post('refund')
  @UseGuards(RefundGuard)
  refund(): { ok: true } {
    return { ok: true };
  }
}
```

`GET /health` على متحكم آخر بلا هذه الزينة يبقى مفتوحًا. `POST /tickets/refund` يعمل `IdentifyDeskGuard` ثم `RefundGuard`. الاسم `lead` يكمل إلى الأنبوب. اسم `guest` يرجع `403` قبل فحص جسم الاسترداد.

## ماذا تتأكد منه

- لا `APP_GUARD` في هذه الأمثلة حتى لا تُغلق الصحة.
- الهوية تُكتب يسار أي حارس يقرأ `user`، أو على الصنف قبله.
- `seat` لا يُفحص داخل الحارسين.
- فشل الحارس لا يُلتقط بـ `try/catch` في الدالة.
