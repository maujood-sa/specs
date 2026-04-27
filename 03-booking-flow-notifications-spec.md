# 03 - Booking Flow Customer Notifications Spec

Status: draft for review, no implementation included.

## Problem

Customers need automatic notifications throughout the booking flow and during/after service. The current backend sends a few generic FCM notifications directly when payment webhook/status change runs, but there is no durable notification system, template library, scheduling, delivery tracking, multi-channel fallback, or customer notification history.

## Current Behavior

- `User.DeviceToken` stores one push token.
- App updates this token through `MainViewModel.updateUserPushToken`.
- Backend sends FCM through `FirebaseAuthService.SendOrderAcceptedNotification`.
- Webhook sends generic "Order Accepted!" or "Payment Failed!" notifications.
- `ChangeOrderStatus` sends generic status update notification.
- App notifications screen currently renders hardcoded sample offer items after a delay.

## Goals

- Notify customers at the right booking milestones.
- Support push first, WhatsApp/email later through the same event pipeline.
- Store notification history so the app notification screen can show real data.
- Localize messages in Arabic and English.
- Avoid duplicate notifications on webhook retries or repeated admin actions.

## Non-Goals

- Do not build marketing campaigns here.
- Do not replace WhatsApp Business spec; this spec defines the generic notification engine.
- Do not send operational staff alerts here; see operations alert spec.

## Order Lifecycle Events

Create explicit backend events. Suggested initial event list:

- `order.created_pending_payment`
- `payment.succeeded`
- `payment.failed`
- `order.confirmed`
- `provider.assigned`
- `provider.accepted`
- `provider.rejected`
- `service.reminder_24h`
- `service.reminder_2h`
- `provider.en_route`
- `provider.arrived`
- `service.started`
- `service.completed`
- `order.cancelled`
- `order.refunded`
- `rating.requested`

Some events do not exist in current code yet. They should be added only when the corresponding business state exists.

## Notification Channels

Phase 1:

- Push notification via FCM/APNs through Firebase.
- In-app notification history.

Phase 2:

- WhatsApp transactional templates.
- Email receipts/status updates.

Phase 3:

- SMS fallback only for critical messages if WhatsApp/push fail.

## Data Model Requirements

Add:

- `NotificationTemplate`
  - `Key`
  - `Channel`
  - `Language`
  - `Title`
  - `Body`
  - `TemplateExternalId`
  - `VariablesJson`
  - `IsActive`

- `Notification`
  - `Id`
  - `UserId`
  - `OrderId`
  - `EventKey`
  - `Title`
  - `Body`
  - `DataJson`
  - `Language`
  - `ReadOn`
  - `CreatedOn`

- `NotificationDispatch`
  - `NotificationId`
  - `Channel`
  - `Recipient`
  - `Provider`
  - `ExternalMessageId`
  - `Status`
  - `AttemptCount`
  - `LastError`
  - `NextRetryOn`

- `DeviceToken`
  - `UserId`
  - `Token`
  - `Platform`
  - `AppVersion`
  - `LastSeenOn`
  - `IsActive`

Replace the single `User.DeviceToken` dependency over time.

## Template Requirements

Templates should include:

- Arabic and English versions.
- Customer first name if available.
- Order number.
- Service name.
- Date/time in Riyadh timezone.
- Provider name when assigned.
- CTA deep link payload: order details screen.

Avoid:

- Full address in push notification body.
- Sensitive vehicle plate in notification body.
- Over-promising arrival times if the provider app cannot update ETA.

## Scheduling Requirements

Scheduled reminders:

- 24 hours before service, if order is paid and not cancelled/completed.
- 2 hours before service, same condition.
- Optional same-day reminder if booking was made within 24 hours.

Need a background job runner:

- Hangfire, Quartz, hosted service, or cloud scheduler.
- Jobs must be idempotent.
- Jobs must re-check order state before sending.

## API Requirements

Mobile:

- `GET /api/v1/notifications`
- `PUT /api/v1/notifications/{id}/read`
- `PUT /api/v1/notifications/read-all`
- `DELETE /api/v1/device-tokens/{id}` or token deactivate on logout.

Backend internal:

- `NotificationService.Enqueue(eventKey, orderId, userId, payload)`
- `NotificationDispatcher.ProcessPending()`

## Push Payload Requirements

Each push should include data:

- `type=order`
- `event_key`
- `order_id`
- `deep_link`
- `sent_at`

Use notification title/body for display, and data payload for navigation.

## Delivery and Token Cleanup

- Store FCM response IDs.
- On `UNREGISTERED` or invalid token response, mark token inactive.
- Refresh token timestamps monthly or when the app becomes active.
- Track delivery success/failure rate per channel.

## Acceptance Criteria

- Real notifications appear in the app notifications screen.
- Payment success/failure sends localized push exactly once per payment event.
- Reminders do not send for cancelled/refunded/completed orders.
- Invalid FCM tokens are deactivated.
- Admin can audit what notification was sent, when, and through which channel.

## Risks

- Current payment webhook may retry and create duplicate transaction/notification effects if idempotency is not implemented first.
- Current order statuses are too coarse for provider/service-progress messages.
- Single-token storage will fail for customers with multiple devices.
- WhatsApp templates require approval and cannot be treated like arbitrary push text.

## Owner Decisions

- Which notifications are mandatory at launch?
- Should WhatsApp be opt-in only or default transactional channel?
- What should happen if push fails: no fallback, WhatsApp fallback, or email fallback?

## Official References

- Firebase Cloud Messaging: https://firebase.google.com/docs/cloud-messaging
- FCM Admin SDK send: https://firebase.google.com/docs/cloud-messaging/send/admin-sdk
- FCM token management: https://firebase.google.com/docs/cloud-messaging/manage-tokens

