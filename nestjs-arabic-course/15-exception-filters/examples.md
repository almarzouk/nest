# أمثلة عملية

ثلاثة ملفات على الشباك: استثناء مجال، فلتر يحفظ الرمز، وتسجيل عام. لا يوجد فلتر يثبّت `500`.

## مثال 1 — مقعد مبيع بمفتاح ثابت

```ts
export class SeatTakenException extends HttpException {
  constructor(seat: number) {
    super(
      { statusCode: HttpStatus.CONFLICT, code: 'SEAT_TAKEN', seat },
      HttpStatus.CONFLICT,
    );
  }
}
```

الخدمة ترمي `new SeatTakenException(4)`. من غير فلتر مخصص يخرج الجسم كما هو لأن الأب `HttpException`.

## مثال 2 — فلتر يمسك HTTP فقط على مسار الاسترداد

```ts
@Catch(HttpException)
export class RefundHttpFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();
    const request = host.switchToHttp().getRequest<Request>();
    const status = exception.getStatus();
    const payload = exception.getResponse();
    const body = typeof payload === 'string' ? { statusCode: status, message: payload } : payload;
    response.status(status).json({ ...body, path: request.url });
  }
}
```

`@UseFilters(RefundHttpFilter)` على `refund` فقط. `409` يبقى `409` ويزداد `path`. `TypeError` لا يمسكه هذا `@Catch` فيكمل إلى العام.

## مثال 3 — عام للمجهول ولـ Middleware

```ts
@Catch()
export class DeskExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();
    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      const payload = exception.getResponse();
      response.status(status).json(typeof payload === 'string' ? { statusCode: status, message: payload } : payload);
      return;
    }
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      statusCode: 500,
      message: 'Internal server error',
    });
  }
}
```

`APP_FILTER` يسجّل هذا الصنف. معرف تتبع تالف في Middleware يرجع `400` من هنا. `refund` الذي رمى `SeatTakenException` لا يصل إلى هنا لأن `RefundHttpFilter` أخص.

## ماذا تتأكد منه

- لا `status(500)` داخل فرع `HttpException`.
- `cause` غير مذكور في `json`.
- فلتر الدالة ضيق أو ينسخ الرمز.
- اختبار Middleware لا يتوقع فلتر `TicketsController`.
