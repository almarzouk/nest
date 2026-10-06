# أمثلة عملية

## عملية واحدة

`NestFactory.create` و `listen` ووحدات الكتالوج والفوترة. لا `createMicroservice`.

## استخراج الفوترة

```ts
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.TCP,
  options: { host: '0.0.0.0', port: 3001 },
});
await app.listen();
```

العميل `BILLING_HOST=billing` في Compose و `127.0.0.1` على الجهاز.

## طلب-رد وحدث

```ts
await lastValueFrom(this.billing.send({ cmd: 'invoice.issue' }, { orderId }));
this.billing.emit('invoice.issued', { orderId, invoiceId });
```

`connect()` في `onApplicationBootstrap` إن كان الناقل شرطًا، و `close()` في `onModuleDestroy`.

## طابور لا بث للبريد

`Transport.RMQ` وطابور `invoices` و `durable: true` و `amqplib`. الإقرار بعد `issue` القابلة للتكرار. Redis Pub/Sub لإبطال كاش كل نسخة فقط.

## خطأ عابر

```ts
throw new RpcException({ message: 'order missing' });
```

## ماذا تتأكد منه

1. لا قاعدة يكتبها الطرفان بعد الاستخراج.
2. لا `localhost` بين حاويات.
3. `send` بلا اشتراك لا يرسل.
4. Kafka بـ `groupId` مختلف لكل من يجب أن يرى كل الأحداث.
