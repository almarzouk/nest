# أمثلة عملية

ثلاثة معترضات على الشباك. كل واحد أثر واحد. الترتيب في المثال الثالث متعمد.

## مثال 1 — سجل لا يغيّر الجسم

```ts
@Injectable()
export class TicketLogInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<{ method: string; url: string }>();
    const started = Date.now();
    const label = `${request.method} ${request.url}`;
    return next.handle().pipe(
      tap({
        next: () => console.log(`${label} ${Date.now() - started}ms`),
        error: () => console.log(`${label} failed`),
      }),
    );
  }
}
```

## مثال 2 — غلاف على متحكم التذاكر

```ts
@Injectable()
export class TicketEnvelopeInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<{ data: unknown }> {
    return next.handle().pipe(map((data) => ({ data })));
  }
}
```

`TicketsController` يحمل الغلاف. `list` ترجع `string[]` والعميل يرى `{ data: string[] }`. `HealthController` بلا الزينة.

## مثال 3 — مهلة على التقرير فقط، والسجل أقرب إلى الدالة

```ts
@Controller('reports')
export class SalesReportController {
  @UseInterceptors(ReportTimeoutInterceptor, TicketLogInterceptor)
  @Get('sales')
  sales(): { total: number } {
    return { total: 10 };
  }
}
```

الدخول: المهلة ثم السجل ثم الدالة، لأن الكتابة من اليسار إلى اليمين على المستوى نفسه. الخروج: `tap` السجل يرى `{ total: 10 }` أولًا، ثم تيار المهلة يمرر القيمة إن صدرت في الوقت. `BadRequestException` من أنبوب على الدالة يمر بفرع `error` في السجل ثم يعاد رميه من `catchError` في المهلة لأنه ليس `TimeoutError`.

لا تخزين في هذه الأمثلة. `of` يتخطى الدالة، والتقرير يجب أن يعمل.

## ماذا تتأكد منه

- لا `subscribe` داخل `intercept`.
- `tap` لا ترجع جسمًا بديلًا.
- الغلاف في مكان واحد.
- مهلة التقرير لا تُعمَّم على `POST /tickets`.
