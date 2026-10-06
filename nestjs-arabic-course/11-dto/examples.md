# أمثلة عملية

ثلاثة عقود لنادٍ للقراءة. كل مثال حد واحد: دخل، خرج، أو مخطط لم يُفحص بعد على المسار.

## مثال 1 — عقد تسجيل

```ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateMemberDto {
  @IsString()
  @MinLength(3)
  fullName: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

`POST /members` يأخذ هذا الصنف. `password` مسموح هنا فقط. لا `id` ولا `joinedAt`.

## مثال 2 — عقد رد منفصل

```ts
export class MemberResponseDto {
  id: string;
  fullName: string;
  email: string;
  joinedAt: string;
}

export function toMemberResponse(member: {
  id: string;
  fullName: string;
  email: string;
  joinedAt: string;
  passwordHash: string;
}): MemberResponseDto {
  return {
    id: member.id,
    fullName: member.fullName,
    email: member.email,
    joinedAt: member.joinedAt,
  };
}
```

`passwordHash` موجود على الكيان وغائب عن الدالة. اختبار JSON يتأكد أن المفتاح غير موجود في الناتج.

## مثال 3 — مخطط دعوة بلا ربط

```ts
import { z } from 'zod';

export const inviteSchema = z.object({
  email: z.email(),
  desk: z.number().int().positive(),
});

export type InviteInput = z.infer<typeof inviteSchema>;
```

هذا مصدر حقيقة لمسار دعوة لاحق. لا `@IsEmail` بجانبه. الربط `@Body({ schema: inviteSchema })` و `StandardSchemaValidationPipe` يبقيان في فصل التحقق. `ValidationPipe` وحده لن يقرأ `inviteSchema`.

## ماذا تتأكد منه

- كل مسار عام له صنف دخل أو مخطط واحد، لا الاثنان.
- دالة الرد لا تنشر `password` ولا `passwordHash`.
- استيراد `CreateMemberDto` في المتحكم استيراد قيمة.
- `UpdateMemberDto` إن وُجد يُشتق بـ `PartialType(CreateMemberDto)` من غير نسخ الحقول.
