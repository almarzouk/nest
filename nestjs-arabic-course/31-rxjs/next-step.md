# التمهيد للفصل التالي

التيار صار أداة معترض: قيمة واحدة، `pipe`، وتحويل أو مهلة أو خطأ. الفصل `32-http-client` يخرج من العملية إلى API أخرى.

ستستعمل `HttpModule.register` و `HttpService` من `@nestjs/axios` لأن الدوال تعيد `Observable` فيطبّق عليها ما حفظته. صفحة «HTTP client» في توثيق Nest الحالي تصف أيضًا `@nestjs/http-client` المبني على `fetch` ويعيد وعودًا. الحزمة الأحدث مسار رسمي آخر. هذا الفصل يدرس عميل Axios الذي ما زال متاحًا، ويربطه بـ `timeout` و `catchError` و `firstValueFrom`.

مهلة RxJS لا تقطع مقبس Axios وحدها. ستضبط `timeout` في `register` أيضًا، وتترجم `AxiosError` إلى استثناء HTTP بدل تسريب جسم المنبع.

افتح [`32-http-client`](../32-http-client/README.md) عندما تستطيع شرح لماذا `await next.handle()` لا يعيد JSON.
