# Authentication و Guards و Context

## الهدف من هذا الدرس

إتاحة طلب HTTP للمحلل والحارس، حتى تعرف من ينشئ الكتاب من غير قراءة `req` الخام في كل دالة.

## المشكلة التي يحلها

حارس REST يقرأ `Authorization` من `ExecutionContext` الخاص بـ HTTP. على GraphQL السياق نفسه لا يحمل `switchToHttp().getRequest()` بشكل تعتمد عليه. المحلل الذي يستدعي `req` من وسيط عام لا يراه إلا إذا وضعته في context عند الإقلاع.

## الفكرة ببساطة

`context` في `GraphQLModule.forRoot` يبني كائنًا لكل عملية، وفيه `req` و `res`. `@Context()` يقرأ هذا الكائن في المحلل. الحارس يستعمله عبر `GqlExecutionContext.create(context).getContext()`.

## التعريف التقني

من صفحة البداية:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  context: ({ req, res }) => ({ req, res }),
});
```

في المحلل:

```ts
@Query(() => String)
userAgent(@Context('req') req: Request): string {
  return req.headers['user-agent'] ?? '';
}
```

الحارس:

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class GqlAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context);
    const req = ctx.getContext<{ req: { headers: Record<string, string> } }>().req;
    return Boolean(req.headers.authorization);
  }
}
```

هذا الحارس يتحقق من وجود الترويسة فقط. التحقق من الرمز نفسه يبقى في خدمة الهوية عندما تصلون إليها. على الاشتراكات، شكل context مختلف لأن الاتصال ليس طلب HTTP نفسه. لا تعِد استعمال هذا الحارس على `Subscription` من غير درس الاشتراك.

`@UseGuards(GqlAuthGuard)` على `createBook` أو على المحلل كله.

## Mental Model

```text
POST /graphql
  context({ req, res })
    Guard: GqlExecutionContext -> req.headers
    Mutation: @Context() -> نفس الكائن
```

## أبسط مثال

إنشاء كتاب ينسب المالك من الطلب، لا من المدخل:

```ts
@Mutation(() => Book)
@UseGuards(GqlAuthGuard)
createBook(
  @Args('input') input: CreateBookInput,
  @Context('req') req: { userId: string },
) {
  return this.books.create(input, req.userId);
}
```

افترض أن وسيطًا سابقًا، أو الحارس نفسه، وضع `userId` على `req` بعد التحقق. الحقل لا يوجد في `CreateBookInput`. العميل لا يرسل هوية نفسه داخل `input`.

## شرح المثال

`@Context('req')` يختار خاصية من كائن context. `@Context()` بلا اسم يعيد الكائن كاملًا. الحارس والمُحلّل يريان النتيجة نفسها لـ `context` الخاصة بهذه العملية. إن نسيت دالة `context` في `forRoot`، يكون `req` غائبًا ويسقط الحارس أو يرفض الجميع.

## ماذا يحدث داخليًا؟

لكل طلب HTTP إلى `/graphql` يستدعي Apollo دالة `context` ويمرر لها الطلب والرد. الناتج يُخزَّن مع تنفيذ العملية. الحارس يعمل قبل المحلل إن كان مزينًا على الدالة. `GqlExecutionContext` الغلاف الذي يترجم `ExecutionContext` العام إلى هذا الكائن. `switchToHttp()` على المسار نفسه قد لا يعيد طلب GraphQL الذي تتوقعه، لذلك الصفحة تحيل الحراس إلى `getContext()`.

لا تضع كائن Prisma أو كلمة المرور في context. ضع معرف المستخدم بعد التحقق. context يُبنى لكل طلب، وهو أيضًا المكان الذي سيعيش فيه DataLoader لاحقًا حتى لا يُشارك بين مستخدمين.

## مثال عملي

`books` بلا حارس لأنها كتالوج منشور. `createBook` عليه الحارس. لا تضع `@UseGuards` على الصنف كله ثم تحاول إعفاء القراءة. GraphiQL للتطوير يرسل الترويسة من محرره. إغلاق الواجهة في الإنتاج لا يلغي الترويسة على `POST`.

## أخطاء شائعة

- قراءة المستخدم من `input.userId`.
- حارس HTTP جاهز على المحلل من غير `GqlExecutionContext`.
- تخزين `req` في خاصية على المحلل الـ singleton، فيختلط مستخدمان.
- مشاركة الحارس كما هو مع اشتراك `graphql-ws` الذي لا يملك `req` نفسه.

## متى نستخدمه؟

عندما تعتمد العملية على هوية الطلب: إنشاء، تحديث، أو قراءة مسودة لا تُعرض في `books` العامة.

## متى لا نستخدمه؟

على استعلام الكتالوج المنشور إن كان عامًا. الحارس هناك تكلفة ورفض بلا قصة منتج. وعلى الملفات المرفوعة التي بقيت على REST، فالحارس هناك HTTP عادي.

## مقارنة

| الباب | كيف تصل إلى الطلب |
| --- | --- |
| متحكم REST | `ExecutionContext.switchToHttp()` |
| محلل GraphQL | `context` ثم `@Context` أو `GqlExecutionContext` |
| اشتراك | سياق الاتصال، درس منفصل |

## تمارين

### تمرين سهل

أي خيار في `forRoot` يمرر `req` إلى التنفيذ؟

<details>
<summary>الحل</summary>
`context: ({ req, res }) => ({ req, res })`.
</details>

### تمرين متوسط

لماذا لا تقرأ الحارس عبر `switchToHttp()` هنا؟

<details>
<summary>الحل</summary>
لأن تنفيذ GraphQL يغلّف الطلب داخل سياقه. `GqlExecutionContext.create(context).getContext()` هو باب الصفحة.
</details>

### تمرين متقدم

لماذا context لكل طلب أنسب من خاصية `this.user` على المحلل؟

<details>
<summary>الحل</summary>
المحلل singleton. الخاصية تتغير بين الطلبات المتزامنة. context معزول مع العملية.
</details>

## أسئلة مراجعة

- من يضع `userId` الذي يقرؤه `createBook`؟
- لماذا الهوية ليست حقلًا في `CreateBookInput`؟
- أي استعلام في المثال يبقى بلا حارس؟

## الخلاصة

`context` يسلّم طلب HTTP لهذه العملية. الحارس يفتحه بـ `GqlExecutionContext`. العميل لا يرسل هويته داخل مدخل الكتاب.

## ماذا بعد؟

لمحة عن N+1 وDataLoader على هذا الـ context نفسه، بلا بناء مكتبة كاملة.

## المصادر الرسمية

- NestJS — Quick start (context): https://docs.nestjs.com/graphql/quick-start
- NestJS — Guards: https://docs.nestjs.com/guards
