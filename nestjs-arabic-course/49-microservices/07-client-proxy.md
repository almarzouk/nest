# ClientProxy

## الهدف من هذا الدرس

إرسال طلب أو حدث من عملية Nest إلى أخرى عبر `ClientProxy`، مع `send` البارد و `emit` الساخن والاتصال الكسول.

## المشكلة التي يحلها

استدعاء HTTP يدوي لكل خدمة يكرر التسلسل والمهلة. `ClientProxy` واجهة موحّدة فوق الناقل: `send(pattern, payload)` للطلب-الرد و `emit(pattern, payload)` للحدث. تحصل على النسخة من `ClientsModule.register` برمز تحقنه، أو من `ClientProxyFactory.create`.

## الفكرة ببساطة

سجّل عميلًا باسم `'BILLING'` مع `transport` وخياراته. احقن `@Inject('BILLING')`. `send` يعيد `Observable` باردًا لا يرسل حتى `subscribe` أو `lastValueFrom`. `emit` ساخن ويحاول النشر فورًا. الاتصال لا يُفتح عند البناء. افتحه في `onApplicationBootstrap` بـ `connect()` إن أردت فشل الإقلاع عند غياب الناقل.

## التعريف التقني

```ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'BILLING',
        transport: Transport.TCP,
        options: { port: 3001 },
      },
    ]),
  ],
})
export class CheckoutModule {}
```

```ts
import { Inject, Injectable, OnApplicationBootstrap } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { lastValueFrom } from 'rxjs';

@Injectable()
export class CheckoutService implements OnApplicationBootstrap {
  constructor(@Inject('BILLING') private readonly billing: ClientProxy) {}

  async onApplicationBootstrap(): Promise<void> {
    await this.billing.connect();
  }

  issue(orderId: string): Promise<{ invoiceId: string }> {
    return lastValueFrom(
      this.billing.send<{ invoiceId: string }>({ cmd: 'invoice.issue' }, { orderId }),
    );
  }
}
```

من غير `lastValueFrom` أو اشتراك، `send` لا يرسل. هذا سلوك الـ Observable البارد في التوثيق.

`emit('invoice.issued', payload)` للحدث. التوثيق يصف عائده كـ Observable ساخن يحاول التسليم حتى بلا اشتراك.

`ClientProxyFactory.create` عندما تأتي الخيارات من `ConfigService`. أغلق العميل في `onModuleDestroy` بـ `close()`.

في Docker عنوان TCP هو اسم الخدمة `billing` لا `localhost` إن كان العميل في حاوية أخرى. `localhost` يبقى صحيحًا لعمليتين على الجهاز نفسه.

من NestJS 12.1.1 على TCP، مستمعو `on()` على العميل يبقون حتى `close()` ويعاد لصقهم على المقبس التالي. لا تسجل المستمع داخل كل إعادة اتصال فتتضاعف.

## Mental Model

```text
ClientsModule name BILLING
  send -> بارد -> lastValueFrom -> ناتج
  emit -> ساخن -> حدث
connect في onApplicationBootstrap
close في onModuleDestroy
```

## أبسط مثال

`issue` أعلاه. النمط يطابق `@MessagePattern({ cmd: 'invoice.issue' })`. الناتج `{ invoiceId }` بعد أن يرد المعالج.

## شرح المثال

`name` رمز الحقن مثل أي رمز نصي، فيحتاج `@Inject`. `connect` يؤخر الجاهزية حتى يقوم المقبس، وهو ما يربط هذا الدرس بـ `OnApplicationBootstrap`. إن كان الغياب مقبولًا عند الإقلاع فاترك الاتصال الكسول حتى أول `send`، وتوقع أول طلب أبطأ أو فاشلًا.

## ماذا يحدث داخليًا؟

1. التسجيل يبني مزودًا بالرمز `'BILLING'`.
2. أول استخدام أو `connect` يفتح النقل.
3. `send` يسلسل النمط والحمولة وينتظر الرد على الـ Observable.
4. `lastValueFrom` يأخذ القيمة الأخيرة ويكمل الوعد أو يرفض عند خطأ أو إكمال فارغ.
5. `emit` ينشر دون انتظار معالج الطرف الآخر.
6. `close` عند الإيقاف يحرر المقبس إن استدعيتها.

## مثال عملي

إعداد من بيئة:

```ts
ClientsModule.registerAsync([
  {
    name: 'BILLING',
    useFactory: (config: ConfigService) => ({
      transport: Transport.TCP,
      options: {
        host: config.getOrThrow<string>('BILLING_HOST'),
        port: config.getOrThrow<number>('BILLING_PORT'),
      },
    }),
    inject: [ConfigService],
  },
])
```

`BILLING_HOST=billing` في Compose و `127.0.0.1` على جهازك. لا تثبت `localhost` في الصورة.

## أخطاء شائعة

- `send` بلا اشتراك فلا رسالة.
- `localhost` من حاوية إلى حاوية.
- نسيان `close` فيترك مقبضًا بعد `app.close()`.
- حقن العميل في كيان الدومين.
- تسجيل `on()` داخل مسار الطلب على TCP فيتضاعف بعد 12.1.1.

## متى نستخدمه؟

من عملية عميل نحو مستمع `@MessagePattern` أو `@EventPattern`.

## متى لا نستخدمه؟

نداء داخل نفس العملية. احقن الخدمة. ولا `send` لأثر لا تحتاج ناتجه.

## مقارنة

| الدالة | Observable | متى تُرسل |
| --- | --- | --- |
| `send` | بارد | عند الاشتراك |
| `emit` | ساخن | فور النداء |
| `connect` | وعد | عند استدعائك له |

## تمارين

### تمرين سهل

لماذا `lastValueFrom` حول `send`؟

<details>
<summary>الحل</summary>

لأن `send` لا يرسل حتى يُشترك في الـ Observable البارد، والوعد أسهل لـ `async`.

</details>

### تمرين متوسط

أين تفتح الاتصال إن أردت منع `listen` قبل الناقل؟

<details>
<summary>الحل</summary>

`await client.connect()` داخل `onApplicationBootstrap`.

</details>

### تمرين متقدم

ماذا تضع في `BILLING_HOST` داخل شبكة Compose؟

<details>
<summary>الحل</summary>

اسم خدمة الفوترة مثل `billing`، لا `localhost`، لأن `localhost` هو الحاوية الحالية.

</details>

## أسئلة مراجعة

- ما رمز الحقن في `register`؟
- ما الفرق بين بارد وساخن هنا؟
- متى `ClientProxyFactory`؟

## الخلاصة

`ClientProxy` يرسل نمطًا وحمولة. اشترك في `send`، واستعمل `emit` للحدث، وصل عند الإقلاع إن كان الناقل شرطًا، وأغلق عند الإيقاف، واستعمل اسم الخدمة بين الحاويات.

## ماذا بعد؟

TCP: النقل الافتراضي بلا وسيط رسالة.

## المصادر الرسمية

- NestJS — Microservices: https://docs.nestjs.com/microservices/basics
