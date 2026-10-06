# أمثلة عملية

ثلاث سلاسل على الكتالوج. الخدمة تبقى `async`. التيار على المعترض فقط.

## مثال 1 — غلاف بلا ملف متدفق

```ts
import { CallHandler, ExecutionContext, NestInterceptor, StreamableFile } from '@nestjs/common';
import { map, Observable, tap } from 'rxjs';

export class CatalogEnvelopeInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const started = Date.now();
    return next.handle().pipe(
      tap(() => console.log(`catalog ${Date.now() - started}ms`)),
      map((data) => (data instanceof StreamableFile ? data : { data })),
    );
  }
}
```

## مثال 2 — مهلة الوكيل

```ts
import { GatewayTimeoutException, HttpException } from '@nestjs/common';
import { TimeoutError, catchError, throwError, timeout } from 'rxjs';

export function proxyTimeout(ms: number) {
  return (source: import('rxjs').Observable<unknown>) =>
    source.pipe(
      timeout(ms),
      catchError((error: unknown) => {
        if (error instanceof TimeoutError) {
          return throwError(() => new GatewayTimeoutException('catalog timed out'));
        }
        if (error instanceof HttpException) {
          return throwError(() => error);
        }
        return throwError(() => error);
      }),
    );
}
```

الاستعمال: `return next.handle().pipe(proxyTimeout(800))` على مسار البحث الخارجي لا على الرفع.

## مثال 3 — اختبار بمشترك واحد

```ts
import { of, lastValueFrom } from 'rxjs';

const result = await lastValueFrom(
  interceptor.intercept(context, {
    handle: () => of({ title: 'NestJS in Practice' }),
  }),
);
```

الناتج المتوقع `{ data: { title: 'NestJS in Practice' } }`. لا `subscribe` إضافي في الاختبار نفسه.

## ماذا تتأكد منه

- لا `mergeMap` ولا `retry` في هذه الملفات.
- `tap` لا يطبع الجسم.
- المهلة ليست عالمية على `AppModule`.
