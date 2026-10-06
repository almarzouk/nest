# ماذا بعد GraphQL المتقدم؟

حماية المخطط صارت طبقات منفصلة: DataLoader لكل طلب ضد N+1، وإضافة `graphql-query-complexity` للوزن، وقاعدة عمق تضيفها أنت، وقائمة استعلامات من البناء، وFederation ببوابة و `ApolloFederationDriver` عندما يتوزع النوع. لا طبقة منها ديكور سحري يغني عن الباقي.

العميل الذي يرسل هذه الاستعلامات ما زال يحتاج هوية أعقد من كلمة مرور رف: مزودًا خارجيًا، رمزًا يدور، ورمزًا يُبطل.

الفصل التالي: OAuth 2.0 و OpenID Connect وتسجيل Google ضمن `@nestjs/authentication`، من غير القول إن وصفة Passport أُزيلت.

[Authentication المتقدم](../58-advanced-authentication/README.md)
