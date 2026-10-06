# التمهيد للفصل التالي

المخطط صار عقد قراءة وكتابة على `/graphql`، والاشتراك قناة GraphQL تحديدًا. الفصل `30-websockets` يفتح مقبسًا لا يمر على حقول SDL: بوابة Nest تتلقى أحداثًا بأسماء أنت تختارها.

## ماذا يبقى هنا

- Code-first مع `autoSchemaFile` و `sortSchema` و `ApolloDriver`.
- GraphiQL خارج الإنتاج، بلا `playground` ككتابة جديدة.
- المحلل في `providers` ويستدعي الخدمة.
- `context` لكل طلب للحارس ولأي DataLoader لاحق.
- N+1 مسمّاة، والعمق في `57-graphql-advanced`.
- الاشتراك بمفتاح `graphql-ws` بعد استعلام الحالة الحالية.
- رفع الملفات لم ينتقل إلى المخطط.

## ماذا ستفرّق هناك

`@WebSocketGateway` صنف مزوّد مثل المحلل، لكن الرسالة `@SubscribeMessage` ليست حقل `Subscription`. Socket.IO هو الافتراض. خطافات الاتصال، والمصادقة على المصافحة، والحراس والأنابيب تعمل على البوابة بشكل مختلف عن HTTP.

لا تبنِ دردشة المتجر داخل `PubSub` الخاص بـ GraphQL إن كان العميل لا يتحدث GraphQL أصلًا.

افتح [`30-websockets`](../30-websockets/README.md).
