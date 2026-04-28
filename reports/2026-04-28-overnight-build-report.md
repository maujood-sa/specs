# 2026-04-28 Overnight Build Report

Status: pushed for owner review.

## Executive Summary

Overnight work focused on low-risk foundations that did not require new business decisions or missing third-party dashboard access:

- Payment/webhook hardening.
- Backend order CSV export and filter/range controls.
- OTP/auth reliability foundations.
- OTP analytics instrumentation.
- Admin-configurable notification and operations alert foundations.
- Repository hygiene and GitHub Actions CI.
- App build verification and environment documentation.
- App payment safety guard.
- iOS source phone-auth shim restoration.

No promocodes, packages, Salla integration, WhatsApp integration, marketing pixels, live activities, or actual operations alert delivery were implemented because those still need product/account decisions or external credentials.

## Repositories Updated

### `maujood-sa/maujood-backend`

Commits:

- `6df8193` - Harden webhooks and add order export
- `fa20c1b` - Add EF local tool manifest
- `468888d` - Clean backend artifacts and add CI
- `b89064a` - Add configurable order export filters
- `1bb28bf` - Add admin notification configuration foundations

Main changes:

- Added `WebhookInbox` model, EF context mapping, migration, and snapshot entry.
- Moyasar webhook endpoint now stores received payloads and skips duplicate event IDs.
- Webhook processing now marks inbox rows as `processing`, `processed`, `failed`, or `unauthorized`.
- Fixed failed-payment handling to create failed purchase transactions.
- Added handling for `payment_failed` and `payment_refunded`.
- Added duplicate transaction guard for the same order/payment/transaction type.
- Added portal CSV export endpoint: `GET /api/v1/Order/export-orders-csv`.
- Added export route alias: `GET /api/v1/Order/export`.
- Added export query filters and configurable max date range through `Exports:Orders:MaxDateRangeDays`.
- Added `AppConfigurationSetting`, `NotificationTemplate`, `NotificationDispatch`, `OperationsAlertRule`, and `OperationsAlertLog` models.
- Added admin configuration endpoints under `GET/POST/PUT /api/v1/Configuration/...` for settings, notification templates, dispatch review, and operations alert rules.
- Added migration `20260428033000_AddNotificationConfiguration`.
- Removed tracked `bin/`, `obj/`, logs, `.DS_Store`, `.csproj.user`, and Firebase service-account JSON from the repo index.
- Added backend GitHub Actions build workflow.
- Added backend Firebase auth diagnostics with masked phone values.
- Hardened Firebase push sends so missing/invalid device tokens do not break order/webhook flows.
- Added `dotnet-tools.json` with `dotnet-ef` 9.0.0.

Required deployment note:

- Apply migration `20260428023000_AddWebhookInbox` before relying on the new webhook inbox in production.
- Apply migration `20260428033000_AddNotificationConfiguration` before using admin notification/configuration endpoints in production.
- Rotate the Firebase service-account key that was previously committed to history, even though it has now been removed from the current tree.

### `maujood-sa/maujood-app`

Commits:

- `5e0d79b` - Document app build JDK requirement
- `b345d1e` - Guard payments against missing Moyasar key
- `10adcb9` - Implement iOS Firebase phone auth shim
- `206a60a` - Track OTP verification analytics
- `5ea0d2e` - Add app CI workflow

Main changes:

- Added `.java-version` set to `17`.
- Updated README build commands and documented the Java 25 Gradle failure mode.
- Added guard so card/STC/Apple Pay flows stop with a user-facing error if `MOYASAR_API_KEY` is blank.
- Replaced the iOS phone-auth runtime stub with Firebase iOS `FIRPhoneAuthProvider` verification and ID-token exchange.
- Added OTP analytics for send, resend, verify, invalid-phone, success, and failure paths.
- OTP analytics uses country code, phone hash, and phone length; it does not send raw phone numbers or OTP codes.
- Added app GitHub Actions workflow on macOS with JDK 17 to compile iOS shared code and build Android debug.

### `maujood-sa/specs`

This report and spec updates:

- Updated OTP spec with iOS phone-auth and backend diagnostics status.
- Updated OTP spec with implemented client analytics events.
- Updated booking notification spec with notification-send guard status.
- Updated booking notification spec with backend template/dispatch configuration foundation.
- Updated admin order export spec with implemented CSV endpoint and remaining gaps.
- Updated admin order export spec with filters, route alias, and max date-range config.
- Updated Moyasar account update spec with webhook/payment hardening status.
- Updated operations alert spec with current prerequisite status and backend rule/log foundation.
- Updated marketing pixels spec to focus the reviewed first scope on Meta, Snapchat, and TikTok.

## Verification Run

Backend:

```bash
dotnet build MaujodService.csproj --no-restore
```

Result:

- Passed.
- Existing warning volume remains high.
- AutoMapper `13.0.1` still has a known high-severity advisory warning.
- EF migration generation required `DOTNET_ROLL_FORWARD=Major` locally because only .NET 10 runtime is installed, while `dotnet-ef` 9 targets .NET 8.

App:

```bash
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :androidApp:assembleDebug --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :appShared:compileKotlinIosArm64 --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :appShared:linkDebugFrameworkIosArm64 --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :androidApp:lintProdRelease --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :appShared:compileKotlinIosArm64 :androidApp:assembleDebug
```

Result:

- Android debug build passed.
- iOS Arm64 shared compile passed.
- iOS debug framework link passed.
- Android production lint passed.
- Combined iOS shared compile + Android debug build passed after OTP analytics changes.

Local environment notes:

- The machine default JDK is Java 25, which fails Gradle/Kotlin script parsing with `IllegalArgumentException: 25.0.1`.
- JDK 17 is required for this project.
- Local Android SDK path was added only to ignored `local.properties`; it was not committed.

## What Is Ready To Review

Backend:

- Whether both export endpoints should remain, or the portal should standardize on `GET /api/v1/Order/export`.
- Whether customer phone, address, and plate number should remain exportable for all `OrderControllerAccess` roles.
- Whether `Exports:Orders:MaxDateRangeDays = 31` is the right synchronous export default.
- Whether webhook inbox raw payload retention is acceptable, or should have retention/encryption/masking rules.
- Whether failed/refunded payment transaction mapping matches operations/finance expectations.
- Whether `ConfigurationControllerAccess` should be admin-only or include operations for alert rule visibility.

App:

- Whether the generic payment error is acceptable when Moyasar publishable key is missing.
- Whether the iOS Firebase phone-auth shim should be tested against the current App Store Firebase project before source-built iOS release.
- Whether build docs should standardize on local `.java-version`, CI JDK pinning, or both.
- Whether OTP analytics event names/properties match your reporting expectations.

Specs:

- Whether order export v1 should remain synchronous CSV or move quickly to async exports.
- Whether customer notifications and operations alerts should be built on an outbox/order-event foundation before any email/WhatsApp work.
- Whether Moyasar account cutover can proceed after staging tests, or needs additional reconciliation tooling first.
- Whether marketing pixels should be implemented as spec-only first for Meta/Snap/TikTok, then backend conversion outbox after consent and account access.

## Remaining High-Priority Work

1. Deploy and verify backend migration `20260428023000_AddWebhookInbox`.
2. Deploy and verify backend migration `20260428033000_AddNotificationConfiguration`.
3. Run real Moyasar webhook tests: paid, failed, duplicate retry, refund, unauthorized token.
4. Run Android/iOS device regression: login OTP, onboarding permissions, home images, booking, payment, order details.
5. Complete Firebase Console verification checklist and compare with OTP analytics.
6. Add export audit log and role-based field policy.
7. Decide email provider/sender and wire operations alert dispatch from paid-order events.
8. Decide AutoMapper advisory strategy: upgrade with licensing review, replace mapping library, or remove AutoMapper usage.

## Known Non-Blocking Warnings

- Backend has many existing nullable and code-analysis warnings.
- Backend dependency warning: AutoMapper `13.0.1`.
- App KMP warnings: redundant source-set edges, expect/actual beta warnings, Compose/Kotlin warnings.
- iOS framework link warning for `nsexception-kt-core` cinterop export.
