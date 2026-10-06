# التمهيد للفصل التالي

التسجيل والحقن يعملان: `providers` تربط الرمز بالوصفة، و `constructor` يطلب الرمز، و `@Injectable()` يبقي Metadata المعاملات. ما زال السؤال مفتوحًا: لماذا هذا أفضل من `new FeesService(new FeeRates())` داخل Controller؟

الفصل `07-dependency-injection` يجيب من جهة التصميم والحاوية. يبدأ بمشكلة `new`: الشبكة المخفية، والنسخ التي لا تستطيع الاختبار استبدالها. ثم DI و IoC و Dependency Inversion كثلاثة أسماء تُخلط كثيرًا. بعد ذلك Container نفسه: أين تعيش الخريطة، وما الفرق بين Registration و Resolution، ولماذا يمسح `import type` المفتاح، وكيف تُبنى الشجرة من الأوراق، ومتى تُشارك نسخة Singleton.

Scope والوصفات الأربع التي رأيتها هنا تبقى كما هي. الفصل التالي يفسر آلية الحل لا أن يعيد كتالوج `useFactory`.

افتح [`07-dependency-injection`](../07-dependency-injection/README.md) بعد أن تستطيع تسجيل Service وحقنها من Module أخرى عبر `exports` من غير النظر إلى الدرس.
