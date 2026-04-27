# 04 - Live Notifications Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants live notifications for service progress. iOS supports Live Activities through ActivityKit. Android now has Live Update notifications on supported Android versions, and older Android versions can use ongoing/progress notifications. The app currently has standard push notification permission handling but no Live Activity, Live Update, or live order session model.

## Current Behavior

- iOS registers for remote notifications in the app delegate.
- Android and iOS collect a push notification token and send it to backend.
- Backend only stores one `User.DeviceToken`.
- Backend has no provider progress states like en-route, arrived, started, or ETA.

## Goals

- Let customers track an active booking from confirmation through service completion without opening the app.
- Support iOS Live Activities for the best iOS experience.
- Support Android Live Updates where available and fall back to a normal ongoing/progress notification elsewhere.
- Keep live state driven by backend order events, not by client guesses.

## Non-Goals

- Do not implement real-time provider tracking until provider apps/integrations can supply location or ETA.
- Do not create a chat system in this feature.
- Do not show sensitive address or vehicle details on lock screen.

## Recommended Phasing

Phase 1 - Backend state foundation:

- Add explicit order progress states:
  - `paid`
  - `provider_assigned`
  - `provider_accepted`
  - `en_route`
  - `arrived`
  - `started`
  - `completed`
  - `cancelled`
- Add event timestamps and actor information.
- Add notification outbox.

Phase 2 - iOS Live Activities:

- Add ActivityKit extension/widgets.
- Start activity after payment success or provider acceptance.
- Update via APNs/FCM Live Activity support or app foreground updates.
- End on completed/cancelled/refunded.

Phase 3 - Android:

- Add Live Update notifications for Android versions/devices that support promotion.
- Add ongoing notification fallback with progress states for older devices.
- Keep deep link to order details.

## Live Session Model

Add `LiveOrderSession`:

- `Id`
- `OrderId`
- `UserId`
- `Platform`
- `DeviceTokenId`
- `ActivityToken`
- `State`
- `StartedOn`
- `LastUpdatedOn`
- `ExpiresOn`
- `EndedOn`
- `LastPayloadJson`

Add `OrderProgressEvent`:

- `OrderId`
- `EventKey`
- `ActorType`: `system`, `admin`, `provider`, `payment_webhook`
- `ActorId`
- `OccurredOn`
- `PayloadJson`

## Live Payload

Safe fields:

- Order number.
- Service name.
- Provider display name.
- Current status label.
- Scheduled time.
- ETA range if available.
- CTA deep link.

Avoid on lock screen:

- Exact address text.
- Phone number.
- Vehicle plate number.
- Payment details.

## iOS Requirements

- Add ActivityKit support and Live Activity entitlement if needed.
- Define `OrderLiveActivityAttributes`.
- Define compact, expanded, and Dynamic Island presentations.
- Capture activity push token and send it to backend tied to order/device.
- Backend can update the activity when order progress changes.
- End the activity cleanly on completion/cancellation.

## Android Requirements

- Use Android Live Updates for active progress if app target/API support and device behavior qualifies.
- Use ongoing notification fallback with stable notification ID per order.
- Update notification when order progress changes.
- Respect user notification permission state.
- Do not require a foreground service unless live location/tracking is actually running.

## Backend Requirements

- Do not update live notifications directly from controllers.
- Controllers/services publish `OrderProgressEvent`.
- Live notification dispatcher consumes events and updates platform-specific sessions.
- Store platform response IDs/errors for debugging.
- Expire live sessions automatically after completion or a maximum lifetime.

## UX Requirements

Suggested statuses:

- "Booking confirmed"
- "Provider assigned"
- "Provider is on the way"
- "Provider arrived"
- "Service in progress"
- "Service complete"
- "Booking cancelled"

Arabic localization must be complete before release.

## Acceptance Criteria

- Paid/active order can start a live session on iOS.
- iOS Live Activity updates after backend status changes.
- Android users see a live/progress notification where supported, or a stable ongoing notification fallback.
- Live session ends on completion/cancellation/refund.
- No sensitive customer data appears on lock screen.
- Backend can inspect session status and last error.

## Risks

- Live Activities are not a substitute for real provider tracking; without provider integration, statuses remain coarse.
- Android support differs by OS version and OEM behavior.
- APNs/FCM Live Activity token handling is easy to get wrong and needs device testing.
- Too many updates can be throttled by platforms.

## Owner Decisions

- Should live tracking begin after payment success or only after provider acceptance?
- Are provider ETAs available manually from operations, or only after provider app integration?
- Is Android live progress required in the first release or acceptable as fallback notification?

## Official References

- Apple ActivityKit: https://developer.apple.com/documentation/activitykit
- Apple Live Activities with push notifications: https://developer.apple.com/documentation/ActivityKit/starting-and-updating-live-activities-with-activitykit-push-notifications
- Android Live Updates: https://developer.android.com/develop/ui/views/notifications/live-update
- FCM iOS Live Activity docs: https://firebase.google.com/docs/cloud-messaging

