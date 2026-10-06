# ماذا بعد المصادقة المتقدمة؟

رف تميّز التفويض (OAuth 2.0) عن الهوية (OIDC و `sub`). Google يمر عبر `OidcService` و `google()` وقرار `OidcAccountResolver`، لا عبر لصق رمز المزود على `/books`. تدوير `TokenService.refresh` يستهلك الرمز، وإعادة المستهلك تبطل العائلة. `revoke` يسقط العائلة أو الجلسة التي يملكها صاحب الإثبات. وصفة Passport ما زالت على https://docs.nestjs.com/recipes/passport لمن يختارها مسارًا وحيدًا.

الخطوة التالية ليست مزودًا آخر. عدة مكتبات على نفس القاعدة، وكل طلب يجب أن يرى كتب مكتبته فقط.

[Multi-tenancy](../59-multi-tenancy/README.md)
