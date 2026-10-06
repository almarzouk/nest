# تمارين الفصل

أجب من غير فتح الدروس، ثم افتح الحل.

## سهل

1. ما المعادل الطويل لـ `providers: [FineService]`؟
2. ماذا يغيّر `@Optional()` على معامل مُنشئ عندما يكون الرمز غير مسجّل؟

<details>
<summary>الحل</summary>

1. `{ provide: FineService, useClass: FineService }`.
2. Nest تمرر `undefined` بدل إيقاف الإقلاع. الصنف يوفّر البديل. الـ Decorator لا يسجل الرمز.

</details>

## متوسط

`VisitContext` عليها `Scope.REQUEST` وتقرأ رأس العضو. `LoansService` افتراضية في الكتابة لكنها تطلب `VisitContext`. `StampClock` عليها `Scope.TRANSIENT` وتُحقن في `AuditService` الافتراضية. ما عمر كل من `LoansService` و `AuditService`، وكم نسخة ساعة لطلبين؟

<details>
<summary>الحل</summary>

- `LoansService` تصبح `REQUEST` لأن العدوى تصعد ممن تحقن.
- `AuditService` تبقى Singleton. `TRANSIENT` لا يغيّر عمر المستهلك.
- ساعة واحدة: تُنشأ عندما تُنشأ `AuditService` عند الإقلاع، والطلبان يريانها.

</details>

## متقدم

التسجيل الآتي يجعل عدّاد `AuditLog` يتفرع رغم أن الفريق أراد نسخة واحدة. أصلح الوصفة الثانية فقط.

```ts
providers: [
  AuditLog,
  { provide: 'DeskAudit', useClass: AuditLog },
]
```

ثم اشرح كيف تحقن واجهة `FeatureFlags` إن أصبح جدول الأعلام `useValue` بدل صنف.

<details>
<summary>الحل</summary>

استبدل الوصفة الثانية بـ `{ provide: 'DeskAudit', useExisting: AuditLog }`. `useClass` كان يبني نسخة في خانة النص منفصلة عن خانة الصنف.

لجدول الأعلام: أعلن `const FEATURE_FLAGS = Symbol('FEATURE_FLAGS')` في ملف مشترك، وسجّل `{ provide: FEATURE_FLAGS, useValue: flags }`، واحقن `@Inject(FEATURE_FLAGS) flags: FeatureFlags`. الواجهة وصف فقط. لا تعِد نداء `Symbol(...)` في ملف الخدمة. الصنف المجرد بديل إن أردت مفتاحًا يُستنتج من نوع المعامل من غير `@Inject`.

</details>
