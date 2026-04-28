# 2026-04-28 Overnight Build Report

Status: pushed for owner review.

## Executive Summary

Overnight work focused on low-risk foundations that did not require new business decisions or missing third-party dashboard access:

- Payment/webhook hardening.
- First backend order CSV export.
- OTP/auth reliability foundations.
- App build verification and environment documentation.
- App payment safety guard.
- iOS source phone-auth shim restoration.

No promocodes, packages, Salla integration, WhatsApp integration, marketing pixels, live activities, or operations alert delivery were implemented because those still need product/account decisions or external credentials.

## Repositories Updated

### `maujood-sa/maujood-backend`

Commits:

- `6df8193` - Harden webhooks and add order export
- `fa20c1b` - Add EF local tool manifest

Main changes:

- Added `WebhookInbox` model, EF context mapping, migration, and snapshot entry.
- Moyasar webhook endpoint now stores received payloads and skips duplicate event IDs.
- Webhook processing now marks inbox rows as `processing`, `processed`, `failed`, or `unauthorized`.
- Fixed failed-payment handling to create failed purchase transactions.
- Added handling for `payment_failed` and `payment_refunded`.
- Added duplicate transaction guard for the same order/payment/transaction type.
- Added portal CSV export endpoint: `GET /api/v1/Order/export-orders-csv`.
- Added backend Firebase auth diagnostics with masked phone values.
- Hardened Firebase push sends so missing/invalid device tokens do not break order/webhook flows.
- Added `dotnet-tools.json` with `dotnet-ef` 9.0.0.

Required deployment note:

- Apply migration `20260428023000_AddWebhookInbox` before relying on the new webhook inbox in production.

### `maujood-sa/maujood-app`

Commits:

- `5e0d79b` - Document app build JDK requirement
- `b345d1e` - Guard payments against missing Moyasar key
- `10adcb9` - Implement iOS Firebase phone auth shim

Main changes:

- Added `.java-version` set to `17`.
- Updated README build commands and documented the Java 25 Gradle failure mode.
- Added guard so card/STC/Apple Pay flows stop with a user-facing error if `MOYASAR_API_KEY` is blank.
- Replaced the iOS phone-auth runtime stub with Firebase iOS `FIRPhoneAuthProvider` verification and ID-token exchange.

### `maujood-sa/specs`

This report and spec updates:

- Updated OTP spec with iOS phone-auth and backend diagnostics status.
- Updated booking notification spec with notification-send guard status.
- Updated admin order export spec with implemented CSV endpoint and remaining gaps.
- Updated Moyasar account update spec with webhook/payment hardening status.
- Updated operations alert spec with current prerequisite status.

## Verification Run

Backend:

```bash
dotnet build MaujodService.csproj --no-restore
```

Result:

- Passed.
- Existing warning volume remains high.
- AutoMapper `13.0.1` still has a known high-severity advisory warning.

App:

```bash
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :androidApp:assembleDebug --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :appShared:compileKotlinIosArm64 --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :appShared:linkDebugFrameworkIosArm64 --no-daemon
JAVA_HOME=/opt/homebrew/Cellar/openjdk@17/17.0.19/libexec/openjdk.jdk/Contents/Home ./gradlew :androidApp:lintProdRelease --no-daemon
```

Result:

- Android debug build passed.
- iOS Arm64 shared compile passed.
- iOS debug framework link passed.
- Android production lint passed.

Local environment notes:

- The machine default JDK is Java 25, which fails Gradle/Kotlin script parsing with `IllegalArgumentException: 25.0.1`.
- JDK 17 is required for this project.
- Local Android SDK path was added only to ignored `local.properties`; it was not committed.

## What Is Ready To Review

Backend:

- Whether `GET /api/v1/Order/export-orders-csv` is acceptable as a first export endpoint name, or should be renamed to the spec target `GET /api/v1/Order/export`.
- Whether the first export can include customer phone, address, and plate number for all admins with `OrderControllerAccess`.
- Whether webhook inbox raw payload retention is acceptable, or should have retention/encryption/masking rules.
- Whether failed/refunded payment transaction mapping matches operations/finance expectations.

App:

- Whether the generic payment error is acceptable when Moyasar publishable key is missing.
- Whether the iOS Firebase phone-auth shim should be tested against the current App Store Firebase project before source-built iOS release.
- Whether build docs should standardize on local `.java-version`, CI JDK pinning, or both.

Specs:

- Whether order export v1 should remain synchronous CSV or move quickly to async filtered exports.
- Whether customer notifications and operations alerts should be built on an outbox/order-event foundation before any email/WhatsApp work.
- Whether Moyasar account cutover can proceed after staging tests, or needs additional reconciliation tooling first.

## Remaining High-Priority Work

1. Deploy and verify backend migration `20260428023000_AddWebhookInbox`.
2. Run real Moyasar webhook tests: paid, failed, duplicate retry, refund, unauthorized token.
3. Run Android/iOS device regression: login OTP, onboarding permissions, home images, booking, payment, order details.
4. Add client OTP analytics events and Firebase Console verification checklist results.
5. Add export filters, audit log, and role-based field policy.
6. Decide operations alert channel/recipients before implementing email/WhatsApp alerts.
7. Clean backend build hygiene later: remove tracked `bin/` and `obj/` artifacts through an explicit cleanup commit.

## Known Non-Blocking Warnings

- Backend has many existing nullable and code-analysis warnings.
- Backend dependency warning: AutoMapper `13.0.1`.
- App KMP warnings: redundant source-set edges, expect/actual beta warnings, Compose/Kotlin warnings.
- iOS framework link warning for `nsexception-kt-core` cinterop export.

