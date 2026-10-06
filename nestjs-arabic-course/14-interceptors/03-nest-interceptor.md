# NestInterceptor

## الهدف من هذا الدرس

كتابة صنف يطبّق `NestInterceptor` ويعرف وسيطَي `intercept`.

## المشكلة التي يحلها

وصف «حول الدالة» لا يكفي للمترجم. الواجهة تثبت اسم الدالة ونوع الرجوع حتى لا يُسمى الصنف معترضًا وهو يطبّق `CanActivate`.

## الفكرة ببساطة

```ts
intercept(context: ExecutionContext, next: CallHandler): Observable<unknown>
```

`context` هو سياق التنفيذ نفسه الذي رآه الحارس. `next` من يستدعي بقية السلسلة. الناتج تيار.

## التعريف التقني

الصنف `@Injectable()` ويطبّق `NestInterceptor<T, R>`. `T` نوع ما يصدره `next.handle()`. `R` نوع ما يصدره التيار الذي ترجعونه من `intercept`.

مثال الصفحة لغلاف الرد:

```ts
export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map((data) => ({ data })));
  }
}
```

هذا الدرس يثبت الواجهة. شرح `handle` وحده في الدرس التالي، وفرق ما قبل النداء وما داخل `pipe` بعده.

`CallHandler` واجهة فيها `handle()`. لا تستورد تيارًا وتتجاهل `next` إلا حين تقصد استبدال التنفيذ.

الربط `@UseInterceptors(TransformInterceptor)` يمرر الصنف للحقن، أو `new` إن لم يوجد اعتماد. العام عبر `useGlobalInterceptors` بلا حقن، أو `APP_INTERCEPTOR` مع `useClass` للحقن. تفاصيل الترتيب في الدرس الأخير.

`intercept` يمكن أن تكون `async` إن احتجت `await` قبل إرجاع التيار. الانتظار قبل `return` ما زال «قبل» التشغيل إن لم تنادِ `handle` داخل الانتظار إلا حين تريد ذلك.

## Mental Model

```text
@Injectable
implements NestInterceptor
intercept(context, next) → Observable
```

الحارس يطبّق واجهة أخرى ويرجع منطقيًا. لا تخلط الاسم `intercept` مع `canActivate`.

## أبسط مثال

```ts
import {
  type CallHandler,
  type ExecutionContext,
  Injectable,
  type NestInterceptor,
} from '@nestjs/common';
import { type Observable } from 'rxjs';

@Injectable()
export class DeskLabelInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<{ url: string }>();
    void request.url;
    return next.handle();
  }
}
```

الصنف صحيح الشكل. يقرأ المسار ثم يعيد تيار البقية بلا `map`. لا أثر ظاهر على JSON. هو هيكل الدرس.

## شرح المثال

- `implements NestInterceptor` يفرض التوقيع.
- `context.switchToHttp()` متاح كما في الحارس. لا تخزّن الطلب على الحقل.
- `return next.handle()` يمرر التيار كما هو. حذف `return` يجعل الناتج `undefined` والإطار لا يملك تيارًا يشترك فيه.
- لا `@UseGuards` على هذا الصنف. الربط زينة معترضات.

## ماذا يحدث داخليًا؟

عند بناء المسار يغلّف Nest المعالج بنسخ المعترضات المرتبطة. عند الطلب، بعد الحراس، تُستدعى `intercept`. القيمة المرجعة يجب أن تكون `Observable`. الاشتراك على أبعد تيار يشغّل `handle` حين تصل السلسلة إليه.

إن رميت استثناء عاديًا قبل `return`، فالتيار لم يُبنَ بعد. الطبقة تعامل الرمي كخطأ طلب. إن رميت داخل `map` فالخطأ على التيار. الفرق عملي حين تضع `catchError` في معترض أبعد: هو يرى أخطاء التيار لا بالضرورة رميًا حدث قبل إرجاع الوصف من معترض داخلي. أبقِ الأخطاء المتوقعة على التيار بـ `throwError` حين تريد التقاطها في `pipe`.

الحقن في الباني يعمل مع تمرير الصنف و `APP_INTERCEPTOR`. `useGlobalInterceptors(new DeskLabelInterceptor())` لا يحقن.

## مثال عملي

`DeskLabelInterceptor` على `TicketsController` يقرأ `url` للتطوير فقط ويعيد `handle`. الاختبار يتأكد أن `GET /tickets` ما زال يرجع جسم الخدمة لأن لا `map`. حين تضيف `map` في درس التغليف، هذا الصنف ليس هو المكان. صنف لكل أثر حتى لا يصبح `intercept` دالة طويلة.

## أخطاء شائعة

- تطبيق `CanActivate` على صنف اسمه Interceptor.
- نسيان إرجاع التيار.
- `subscribe` داخل `intercept` وعدم الإرجاع.
- تخزين `context` على `this`.

## متى نستخدمه؟

لكل التفاف حول مسارات التذاكر أو حول التطبيق، بصنف واحد لكل مهمة: وسم، قياس، غلاف، مهلة.

## متى لا نستخدمه؟

لا تكتب الواجهة على دالة Middleware. التوقيع مختلف. لا ترجع `boolean` من `intercept`.

## مقارنة

| | `NestInterceptor` | `CanActivate` |
| --- | --- | --- |
| الدالة | `intercept` | `canActivate` |
| الوسيط الثاني | `CallHandler` | لا يوجد |
| الرجوع | `Observable` | منطقي أو ما يعادله |

## تمارين

### تمرين سهل

ما اسم الدالة التي تطبّقها الواجهة، وما نوع رجوعها؟

<details>
<summary>الحل</summary>

`intercept`، وترجع `Observable`.

</details>

### تمرين متوسط

ماذا يحدث إن قرأت الطلب ثم انتهت الدالة من غير `return next.handle()`؟

<details>
<summary>الحل</summary>

لا يوجد تيار يرجعه المعترض. البقية لا تُشغَّل بالطريقة التي ينتظرها الإطار، والرد لا يكتمل كنجاح عادي. أرجع ناتج `handle` أو تيارًا بديلًا مقصودًا.

</details>

### تمرين متقدم

المعترض يحتاج `ConfigService` ليقرأ بادئة السجل. أي ربط عام يحقنها؟

<details>
<summary>الحل</summary>

`{ provide: APP_INTERCEPTOR, useClass: DeskLabelInterceptor }` في وحدة ترى الإعداد. `new` داخل `useGlobalInterceptors` لا يحقن.

</details>

## أسئلة مراجعة

- ما وسيطا `intercept`؟
- ماذا يعني المعامل النوعي `R`؟
- كيف تمرر البقية كما هي؟
- أي رمز مزوّد يحقن معترضًا عامًا؟

## الخلاصة

`NestInterceptor` تثبت `intercept(context, next)` وتيارًا مُرجَعًا. السياق للقراءة، و `next` لبقية السلسلة. الصنف يُربَط كحارس من حيث الحقن، ويختلف في الواجهة ونوع الرجوع.

## ماذا بعد؟

الدرس التالي `next.handle`: ماذا يشغّل الاشتراك، وماذا يحدث إن لم تُستدعَ.

## المصادر الرسمية

- [Interceptors](https://docs.nestjs.com/interceptors)
