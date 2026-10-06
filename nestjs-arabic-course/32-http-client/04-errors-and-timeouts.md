# الأخطاء والمهل

## الهدف من هذا الدرس

تحويل فشل Axios إلى استثناء Nest أو إلى `null` مقصود، مع مهلة تقطع الطلب.

## المشكلة التي يحلها

`firstValueFrom` يرفض بـ `AxiosError` يحمل عنوان الناشر وجسمه. المرشح الافتراضي قد يحوّله إلى `500` ويسرب `error.response.data`. وانتهاء `timeout` في Axios رمزه الشائع `ECONNABORTED`، بينما `timeout` في RxJS يعطي `TimeoutError`. من غير ترجمة يرى الزائر مكتبة لا قصة ISBN.

## الفكرة ببساطة

ضع `timeout` في `HttpModule.register` ليُقطع Axios. حول `await` أو داخل `pipe` قبل `firstValueFrom`، صنّف الخطأ: مهلة، غياب عند الناشر، عقد مكسور، ثم عطل. لا تعِد جسم الخطأ الخام.

## التعريف التقني

`AxiosError` من `axios` فيه `code` و `response?.status` و `response?.data`. مهلة Axios تضبط `code` إلى `ECONNABORTED` غالبًا. لا يوجد رد HTTP في هذه الحالة.

خريطة المتجر:

| حالة الصادر | ماذا تعيد `lookup` |
| --- | --- |
| `200` وعنوان نصي | `{ title }` |
| `404` | `null` إن كان الغياب معروفًا |
| مهلة أو بلا رد | ترمي `GatewayTimeoutException` |
| `5xx` أو JSON بلا `title` | ترمي `BadGatewayException` |
| `401` من الناشر | ترمي `BadGatewayException` أيضًا. مفتاحك خاطئ، والزائر لا يصلح ذلك |

لا تترجم `401` الخارجي إلى `401` وارد. الزائر سيظن أن جلسته انتهت.

## Mental Model

```text
AxiosError
  ECONNABORTED أو بلا response -> 504
  status 404                   -> null
  غير ذلك                      -> 502
HttpException معروف            -> يُمرَّر
```

## أبسط مثال

```ts
import { BadGatewayException, GatewayTimeoutException, Injectable } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { AxiosError } from 'axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class IsbnService {
  constructor(private readonly http: HttpService) {}

  async lookup(isbn: string): Promise<{ title: string } | null> {
    try {
      const response = await firstValueFrom(
        this.http.get<{ title?: string }>(`/isbns/${encodeURIComponent(isbn)}`),
      );
      if (typeof response.data?.title !== 'string') {
        throw new BadGatewayException('isbn payload missing title');
      }
      return { title: response.data.title };
    } catch (error) {
      if (error instanceof BadGatewayException || error instanceof GatewayTimeoutException) {
        throw error;
      }
      return this.mapAxios(error);
    }
  }

  private mapAxios(error: unknown): { title: string } | null {
    const axiosError = error as AxiosError;
    if (axiosError.code === 'ECONNABORTED' || !axiosError.response) {
      throw new GatewayTimeoutException('isbn timed out');
    }
    if (axiosError.response.status === 404) {
      return null;
    }
    throw new BadGatewayException('isbn upstream failed');
  }
}
```

## شرح المثال

`catch` يمرر استثناءاتك بعد فحص الحمولة حتى لا يعيد `mapAxios` تغليفها. `404` يرجع `null` فتكمل `BooksService` بعنوان المستخدم. المهلة استثناء لأن الصمت ليس «ISBN غير موجود». الرسالة للزائر ثابتة. تفاصيل `axiosError.response.data` تذهب إلى السجل في فصل التسجيل لا إلى JSON.

## ماذا يحدث داخليًا؟

Axios يشغّل مؤقت `timeout` من `register`. عند النار يُلغى الطلب ويرفض الاشتراك. `firstValueFrom` يرفض الوعد. `mapAxios` لا يرى `TimeoutError` إلا إذا أضفت `timeout()` من RxJS على التيار أيضًا. مهلة واحدة على Axios تكفي هنا حتى لا يتسابق مؤقتان. أضف مؤقت RxJS فقط إن كان هناك تأخير قبل Axios. `404` لا يمر من `try` الناجح لأن Axios يرفض الرموز خارج 2xx افتراضيًا، لذلك يصل عبر `catch`.

## مثال عملي

في `BooksService.create`:

```ts
const fromIsbn = input.isbn ? await this.isbn.lookup(input.isbn) : null;
```

إن رمت `lookup` بسبب المهلة، الإنشاء يتوقف ويصل الزائر `504`. إن أعادت `null`، العنوان من المستخدم. إن أردت عدم إيقاف الإنشاء عند المهلة، التقط `GatewayTimeoutException` في `create` وسجّل ثم أكمل. لا تغيّر `lookup` لتعيد `null` للمهلة، وإلا ضاع الفرق بين «غير موجود» و«لم نعرف».

## أخطاء شائعة

- `catch { return null }` لكل الأخطاء.
- إرجاع `error.response.data` في `HttpException`.
- ترجمة `401` الناشر إلى `UnauthorizedException` للزائر.
- مهلة RxJS ومهلة Axios بقيمتين مختلفتين على المسار نفسه بلا سبب.

## متى نستخدمه؟

على كل دالة `HttpService` تخرج من العملية.

## متى لا نستخدمه؟

داخل مرشح عام يبتلع `AxiosError` من أي وحدة. الترجمة معرفة المنبع، ومكانها `IsbnService`.

## مقارنة

| الرمز عند الناشر | ليس | هو في هذا العقد |
| --- | --- | --- |
| `404` | `NotFoundException` لكتابك | `null` |
| مهلة | `null` | `504` |
| `401` خارجي | جلسة الزائر | `502` |

## تمارين

### تمرين سهل

أي `code` يميز مهلة Axios غالبًا؟

<details>
<summary>الحل</summary>
`ECONNABORTED`.
</details>

### تمرين متوسط

لماذا `401` من الناشر ليس `401` لزائرك؟

<details>
<summary>الحل</summary>
لأن الرمز يخص مفتاح الخادم نحو الناشر. الزائر لا يملك إصلاحه، و `401` الوارد يعني جلسته.
</details>

### تمرين متقدم

ما الذي يضيع إن أعادت المهلة `null` مثل `404`؟

<details>
<summary>الحل</summary>
لن تستطيع الخدمة أن تقرر إيقاف البيع أو إكمال العنوان اليدوي. الغياب والصمت يصيران الشكل نفسه.
</details>

## أسئلة مراجعة

- أين تُضبط مهلة المقبس؟
- ماذا يُسجَّل وماذا يُرمى للعميل؟
- متى يمر `BadGatewayException` من `catch` بلا إعادة تغليف؟

## الخلاصة

صنّف `AxiosError` إلى `null` و `504` و `502`. لا تسرّب جسم الناشر، ولا تخلط مهلة الغياب.

## ماذا بعد؟

تجميع العنوان والمهلة والمفتاح في إعداد غير مثبّت في الكود.

## المصادر الرسمية

- NestJS — HTTP module (Axios): https://docs.nestjs.com/techniques/http-module
- Axios — Handling errors: https://axios-http.com/docs/handling_errors
