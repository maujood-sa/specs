# Maujood Future Work Spec Pack

Status: living spec pack for owner review. Some low-risk foundation items were implemented after the first draft; see the implementation reports below.

This folder contains product and engineering specifications for the future items requested after the codebase handoff. The goal is to make each item concrete enough to review, prioritize, estimate, and then implement safely.

## Current System Snapshot

- Mobile app: Kotlin Multiplatform shared app with Android and iOS shells.
- Backend: ASP.NET Core 8 API with SQL Server, Firebase Admin, Moyasar webhook handling, S3 image uploads, and portal/mobile route groups.
- Admin portal: only backend portal APIs are present in this codebase. No separate admin frontend source was found locally.
- Orders: created as `Scheduled` with an initial pending online transaction. Payment success/failure is recorded later by Moyasar webhook.
- Payment: mobile app uses Moyasar SDKs and gets the publishable key from Firebase Remote Config.
- Notifications: backend can send direct FCM notifications to a single stored `User.DeviceToken`, but there is no notification outbox, template system, delivery log, or notification history API.
- Providers: backend stores provider products, availability windows, and polygon service areas. There is no provider-facing API contract yet.
- Promotions/packages/Salla/WhatsApp/marketing pixels: not currently modeled as first-class backend concepts.

## Implementation Reports

- [2026-04-28 Overnight Build Report](./reports/2026-04-28-overnight-build-report.md)

## Implemented Foundation Since First Draft

- Backend Moyasar webhook inbox and duplicate webhook guard.
- Backend payment failure/refund handling improvements.
- Backend synchronous CSV order export endpoint.
- Backend Firebase auth diagnostics with masked phone logging.
- Backend Firebase push notification send guards.
- App payment guard for missing Moyasar publishable key.
- App iOS Firebase phone-auth compatibility shim.
- App/backend build-tooling documentation updates.

## Shared Architecture Recommendation

Several future items depend on the same missing primitives. Before implementing them independently, define these shared foundations:

1. `OrderEvent` domain events for order lifecycle changes.
2. `NotificationTemplate` and `NotificationDispatch` tables for customer and operations messaging.
3. `IntegrationEventOutbox` for reliable outbound calls to WhatsApp, marketing pixels, providers, Salla, email, and push.
4. Idempotency keys on all payment, webhook, provider, and Salla entry points.
5. A stable `OrderChannel` field (`mobile_app`, `admin_portal`, `salla`, `provider_api`, etc.).
6. Audit logs for admin actions and financial-affecting changes.
7. Clear Saudi/Riyadh timezone handling instead of `DateTime.UtcNow.AddHours(3)` scattered in business logic.

## Spec Index

| # | Spec | Primary Decision Needed |
|---|---|---|
| 01 | [OTP Reliability](./01-otp-reliability-spec.md) | Keep Firebase-only OTP or add fallback SMS/WhatsApp provider. |
| 02 | [Marketing Pixels](./02-marketing-pixels-spec.md) | Web-only first, or web + app + server-side events together. |
| 03 | [Booking Flow Customer Notifications](./03-booking-flow-notifications-spec.md) | Which order states are real business milestones. |
| 04 | [Live Notifications](./04-live-notifications-spec.md) | iOS Live Activities first, Android Live Updates/ongoing notification later. |
| 05 | [Promocodes](./05-promocodes-spec.md) | Simple discounts first or full rule engine. |
| 06 | [Packages and Wallet Credit](./06-packages-wallet-spec.md) | Service bundles, wallet credit, or both. |
| 07 | [Provider App Integrations](./07-provider-app-integrations-spec.md) | One standard provider API vs custom integrations per provider. |
| 08 | [Salla Website Booking Integration](./08-salla-website-booking-integration-spec.md) | Salla checkout as source or Maujood backend as source. |
| 09 | [Admin Order Export](./09-admin-order-export-spec.md) | Synchronous CSV/XLSX first or async export jobs. |
| 10 | [WhatsApp Business Transactional Messaging](./10-whatsapp-business-spec.md) | Cloud API direct vs provider/BSP. |
| 11 | [Moyasar Account Update](./11-moyasar-account-update-spec.md) | Planned cutover date and key ownership. |
| 12 | [Operations New Order Alerts](./12-operations-new-order-alerts-spec.md) | Email only first or email + WhatsApp escalation. |

## Review Method

For each spec, review these sections first:

- "Owner decisions" lists business choices that should be made before implementation.
- "Acceptance criteria" defines when implementation should be considered done.
- "Risks" calls out the areas most likely to cause delays, support load, or incorrect financial behavior.

## Suggested Implementation Order

Recommended order after review:

1. Finish observability and idempotency foundations: client OTP diagnostics, order events, notification outbox. Webhook inbox/payment idempotency and backend auth diagnostics are now partially implemented.
2. Add customer and operations notifications on top of the order-event foundation.
3. Expand order export with filters/audit/async jobs. A first synchronous CSV backend endpoint is now implemented.
4. Add Salla integration only after deciding the website source-of-truth model.
5. Add promocodes, then packages, after pricing fields are normalized.
6. Add provider integrations and live notifications after order progress states are explicit.
7. Migrate/update Moyasar account only after webhook idempotency and payment failure handling are hardened.

## Official References Used

- Firebase Auth phone sign-in: https://firebase.google.com/docs/auth/android/phone-auth
- Firebase Auth limits: https://firebase.google.com/docs/auth/limits
- Firebase Cloud Messaging: https://firebase.google.com/docs/cloud-messaging
- FCM token management: https://firebase.google.com/docs/cloud-messaging/manage-tokens
- FCM Admin SDK send: https://firebase.google.com/docs/cloud-messaging/send/admin-sdk
- Apple ActivityKit: https://developer.apple.com/documentation/activitykit
- Android Live Updates: https://developer.android.com/develop/ui/views/notifications/live-update
- Google Analytics ecommerce events: https://developers.google.com/analytics/devguides/collection/ga4/ecommerce
- Meta Pixel: https://developers.facebook.com/docs/meta-pixel/get-started
- Meta Conversions API: https://developers.facebook.com/docs/marketing-api/conversions-api
- WhatsApp Cloud API: https://developers.facebook.com/docs/whatsapp/cloud-api
- Salla authorization and webhooks: https://docs.salla.dev/421118m0 and https://docs.salla.dev/421119m0
- Moyasar docs and webhooks: https://docs.moyasar.com/ and https://docs.moyasar.com/guides/dashboard/setting-up-webhooks/
