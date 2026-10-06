# التمهيد للفصل التالي

المسارات الآن تستقبل الطلب وتعيد قيمة. `ShelvesController` يستدعي `this.shelves.findAll()`، لكننا لم نبنِ بعد مكان القاعدة ولا طريقة تسجيله.

الفصل `06-providers` يعلّم Provider: الصنف الذي يُحقَن، وعلى رأسه Service حيث تعيش قاعدة العمل. `@Injectable()` يعلن أن الحاوية تستطيع إدارة الصنف، ومصفوفة `providers` في Module هي التسجيل. `constructor(private readonly shelves: ShelvesService)` الذي رأيته هنا يصبح حقنًا حقيقيًا عندما تكون الخدمة مسجّلة.

سترى أيضًا Scope بنظرة أولى: Singleton هو الافتراضي، و `REQUEST` يعدي من يعتمد عليه ويثقل الطلب، و `TRANSIENT` نسخة لكل موضع حقن. ثم أشكال التسجيل `useClass` و `useValue` و `useFactory` و `useExisting`، والرمز الذي تُحقن به عندما لا يكون الرمز صنفًا.

أبقِ Controller رفيعًا كما في هذا الفصل. انقل القرار إلى Service، ثم افتح [`06-providers`](../06-providers/README.md).
