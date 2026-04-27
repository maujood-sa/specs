# 01 - OTP Reliability Spec

Status: draft for review, no implementation included.

## Problem

Customers sometimes do not receive OTP SMS codes. The current mobile app uses Firebase Phone Authentication on the client, then sends the Firebase ID token to the backend after successful OTP verification. The backend does not send OTPs itself.

Because Firebase Phone Auth is client-side, the backend cannot currently see the OTP send attempt, Firebase callback type, Firebase error code, delivery delay, or whether the issue came from app verification, quota, regional policy, carrier filtering, or user input.

## Current Behavior

- Login screen validates a Saudi phone number and formats it as E.164.
- Android calls Firebase `PhoneAuthProvider.verifyPhoneNumber`.
- On code sent or automatic verification, the app navigates to verification screen.
- On OTP entry, app signs in to Firebase and sends the Firebase ID token to backend `/User/authenticate`.
- Backend verifies the Firebase token with Firebase Admin SDK and creates or returns a Maujood user.
- iOS App Store build works today, but the current source compatibility shim does not yet implement iOS phone verification.

Relevant current code:

- `maujood-app-main/appShared/src/commonMain/kotlin/com/maujood/app/presentation/auth/login/LoginViewModel.kt`
- `maujood-app-main/appShared/src/commonMain/kotlin/com/maujood/app/presentation/auth/verification/VerificationViewModel.kt`
- `maujood-app-main/coreCompat/src/androidMain/kotlin/com/metacto/strapikmm/repos/PhoneAuthClient.android.kt`
- `maujood-backend-main/Services/Maujod/Implementation/UserService.cs`

## Goals

- Make OTP reliability diagnosable.
- Reduce customer support ambiguity when an OTP is not delivered.
- Identify whether Google/Firebase console setup is the problem.
- Provide a reviewed fallback plan if Firebase SMS remains unreliable.

## Non-Goals

- Do not replace Firebase Auth immediately.
- Do not add WhatsApp OTP until legal, product, and anti-abuse decisions are approved.
- Do not log raw OTP codes.
- Do not store full phone numbers in analytics without masking/hashing.

## Firebase Console Checklist

Verify these before changing code:

- Firebase Authentication phone provider is enabled.
- SMS region policy allows Saudi Arabia.
- Billing plan supports production SMS volume. Firebase documentation says verification SMS is Blaze-only for Firebase Authentication and has limits.
- Android app SHA-1 and SHA-256 fingerprints are correct for the exact production signing certificate.
- Android package name matches the app in Firebase.
- `google-services.json` is for the intended Firebase project and app.
- iOS bundle ID, APNs auth key/certificate, and reversed client IDs are correct.
- Firebase Auth Activity Logging is enabled if available, to inspect codes sent per phone number.
- Google Cloud API key restrictions allow required Firebase Auth/reCAPTCHA flows.
- Play Integrity requirements are satisfied for Android production builds.

## Instrumentation Requirements

Add client analytics events:

- `otp_send_started`
- `otp_code_sent`
- `otp_auto_verified`
- `otp_send_failed`
- `otp_resend_started`
- `otp_resend_failed`
- `otp_verify_started`
- `otp_verify_failed`
- `otp_login_success`

Event fields:

- platform: `android` or `ios`
- app version and build flavor
- masked phone country and last two digits only
- firebase project id, if safe to expose internally
- Firebase exception class and error code
- verification duration in milliseconds
- resend count in current session
- current auth locale
- whether Play Store install/source is available on Android

Add backend auth diagnostics:

- Log `/User/authenticate` success/failure with Firebase UID, masked phone, app version, and request correlation ID.
- Add a support-only endpoint/report for recent auth attempts by masked phone, without exposing secrets or raw tokens.

## Product UX Requirements

- Show a specific retry state when Firebase returns rate limit, invalid phone, region blocked, or internal error.
- Do not leave user stuck on verification screen if no SMS arrived.
- Provide resend timer and limit messaging.
- Add a "Try again later" support path after repeated failures.
- Consider offering support contact/WhatsApp chat after repeated OTP send failures.

## Fallback Options

### Option A - Firebase-only with diagnostics

Lowest implementation risk. Best first step.

- Keep Firebase Phone Auth.
- Add instrumentation, console fixes, and support tools.
- Use fictional Firebase test numbers for QA without consuming SMS quota.

### Option B - Add SMS provider fallback

More reliable operationally, more security work.

- Backend owns OTP creation and verification for fallback path.
- Use a Saudi-friendly SMS provider.
- Add rate limits, device fingerprinting, hashed OTP storage, expiry, attempt limits, and audit logs.
- Link fallback-verified phone number to existing Firebase or backend identity carefully.

### Option C - Add WhatsApp OTP fallback

Best for customers who prefer WhatsApp, but requires WhatsApp Business template approval and opt-in handling.

- Use approved WhatsApp authentication/utility templates.
- Same backend OTP security requirements as SMS fallback.
- Do not mix marketing messages into OTP flow.

## Proposed Phases

Phase 1:

- Verify Firebase console settings and signing fingerprints.
- Add client and backend diagnostics.
- Add clearer UI errors and support-friendly logs.

Phase 2:

- Build OTP health dashboard: sends, code-sent rate, verify rate, resend rate, error-code breakdown, time-to-code-sent.
- Compare Android vs iOS and production vs QA.

Phase 3:

- If failures remain above acceptable threshold, implement backend-owned fallback provider.

## Acceptance Criteria

- Support can distinguish "Firebase rejected send", "code sent but not delivered", "wrong OTP", and "backend auth failed".
- OTP attempts are visible by platform and version.
- Saudi-region configuration and Android fingerprints are verified.
- No raw OTP or full phone number appears in logs or analytics.
- Customer sees useful retry/support states instead of generic failure.

## Risks

- Firebase anti-abuse limits can change without notice.
- Carrier filtering can produce no visible Firebase error.
- Adding a fallback OTP path increases fraud and account-takeover risk if not rate-limited well.
- iOS source OTP implementation must be restored before source-built iOS releases can rely on this plan.

## Owner Decisions

- What is the acceptable OTP failure rate before adding fallback SMS/WhatsApp?
- Should customer support be allowed to manually verify accounts? Recommended: no, except through a separate audited recovery flow.
- Is WhatsApp acceptable as a login/OTP channel, or only as transactional messaging?

## Official References

- Firebase Android phone auth: https://firebase.google.com/docs/auth/android/phone-auth
- Firebase Auth limits: https://firebase.google.com/docs/auth/limits

