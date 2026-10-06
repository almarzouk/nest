# أمثلة عملية

## علم عام يغلب على الدالة

```ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```ts
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),
  context.getClass(),
]);
```

الدالة أولًا. `@Public()` على `GET /health` يسمح حتى لو بقي المتحكم بلا العلم. غياب المفتاح يعني أن الحارس يكمل فحص الجلسة.

## أدوار تُستبدل لا تُدمج

```ts
@Controller('invoices')
@Roles('staff')
export class InvoicesController {
  @Post()
  @Roles('payroll')
  create(): { ok: true } {
    return { ok: true };
  }
}
```

مع `getAllAndOverride` يرى `POST` قائمة `payroll` فقط. مع `getAllAndMerge` يرى القائمتين. اختر قاعدة واحدة في الحارس.

## ماسح أعلام عند الإقلاع

```ts
const tagged = this.discovery.getProviders({
  metadataKey: FeatureFlag.KEY,
});
```

`FlagsModule` يستورد `DiscoveryModule`. المسح في `onApplicationBootstrap` بعد اكتمال الرسم. لا تستدعِ المسح من داخل `GET`.

## فاعل حسب القناة

```ts
if (context.getType() === 'rpc') {
  return context.switchToRpc().getData<{ userId?: string }>().userId;
}
return context.switchToHttp().getRequest<{ user?: { id: string } }>().user?.id;
```

الخدمة تستقبل `string` الناتج، لا السياق.

## ماذا تتأكد منه

1. مفتاح المزخرف يطابق مفتاح `Reflector`.
2. `APP_GUARD` أو `@UseGuards` موجود فعلًا وإلا فاللصق لا يمنع أحدًا.
3. `DiscoveryModule` في `imports` قبل حقن الماسح.
4. حارس التطبيق الهجين يفرع على `getType()` قبل `switchToHttp`.
