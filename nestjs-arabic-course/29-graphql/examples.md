# أمثلة عملية

ثلاثة عمليات على الكتالوج. الجذر يستخدم code-first و Apollo. الاستيراد المحلي ينتهي بـ `.js`.

## مثال 1 — الإقلاع والمخطط

```ts
import { Module } from '@nestjs/common';
import { join } from 'node:path';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { GraphQLModule } from '@nestjs/graphql';
import { BooksModule } from './books/books.module.js';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
      sortSchema: true,
      context: ({ req, res }) => ({ req, res }),
      subscriptions: { 'graphql-ws': true },
    }),
    BooksModule,
  ],
})
export class AppModule {}
```

لا `typePaths` هنا. لا مفتاح `playground`. GraphiQL يعمل خارج `production` وحده.

## مثال 2 — قراءة وكتابة

`book.model.ts` و `create-book.input.ts` كما في درس الأنواع. المحلل:

```ts
@Resolver(() => Book)
export class BooksResolver {
  constructor(
    private readonly books: BooksService,
    private readonly authors: AuthorsService,
  ) {}

  @Query(() => [Book])
  books() {
    return this.books.findPublished();
  }

  @Mutation(() => Book)
  @UseGuards(GqlAuthGuard)
  createBook(
    @Args('input') input: CreateBookInput,
    @Context('req') req: { userId: string },
  ) {
    return this.books.create(input, req.userId);
  }

  @ResolveField(() => Author)
  author(@Parent() book: { authorId: string }) {
    return this.authors.findById(book.authorId);
  }
}
```

عملية البطاقة لا تطلب `author` فلا يُستدعى الحقل. عملية التفاصيل تطلبه وتكشف N+1 على القائمة، وذلك قياس لا محمّل عام.

## مثال 3 — اشتراك بعد الحفظ

```ts
@Subscription(() => Book)
bookPublished() {
  return this.events.asyncIterator('book.published');
}
```

بعد `books.create` الناجح:

```ts
await this.events.publish('book.published', { bookPublished: saved });
```

العميل يفتح الاستعلام أولًا ثم يترك الاشتراك مفتوحًا في GraphiQL.

## ماذا تتأكد منه

- `schema.gql` يتغير فقط عندما تتغير الزينة، وبترتيب ثابت مع `sortSchema`.
- `CreateBookInput` بلا `userId`.
- ملف الغلاف ما زال على REST.
- حمولة النشر شكل `Book` لا كيان Prisma.
