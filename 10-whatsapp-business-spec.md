# 10 - WhatsApp Business Transactional Messaging Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants WhatsApp Business for transactional messages. The backend currently has no WhatsApp integration, no message template registry, no opt-in tracking, no status webhook receiver, and no multi-channel notification dispatch system.

## Current Behavior

- Customer notifications are direct push only.
- Backend does not store notification history or external message status.
- Phone numbers exist on `User.MobileNumber`.
- OTP currently uses Firebase, not WhatsApp.

## Goals

- Send approved transactional WhatsApp messages for booking/payment/service updates.
- Track delivery status.
- Use the same notification event/template model as push/email.
- Keep WhatsApp OTP separate from transactional messaging unless later approved.

## Non-Goals

- Do not send marketing broadcasts in this feature.
- Do not use unofficial WhatsApp automation or browser sessions.
- Do not replace Firebase OTP in this phase.
- Do not send free-form business-initiated messages without template approval.

## Integration Choice

### Option A - Meta WhatsApp Cloud API direct

Pros:

- Direct control.
- No extra BSP layer.
- Official Meta API.

Cons:

- Requires Meta app, business verification, permanent token setup, webhook handling, template management.

### Option B - Business Solution Provider (BSP)

Pros:

- Faster onboarding/support.
- Some providers simplify templates, logs, and billing.

Cons:

- Extra cost and vendor dependency.
- API may differ from official Meta API.

Recommended v1: choose based on owner access to Meta Business Manager. If Meta access is clean, use Cloud API direct. If access/business verification is blocked, use a BSP.

## Required Business Setup

- Meta Business Manager ownership confirmed.
- WhatsApp Business Account connected.
- Phone number available for Cloud API/BSP use.
- Display name approved.
- Permanent system user token or provider credentials.
- Webhook callback URL.
- Message templates in Arabic and English approved.
- Customer opt-in language added to app/web/privacy policy.

## Data Model Requirements

Reuse notification models from the booking notification spec, plus:

Add `WhatsAppTemplate`:

- `TemplateKey`
- `MetaTemplateName`
- `Language`
- `Category`: `utility`, `authentication`, `marketing`
- `BodyVariables`
- `Status`: `draft`, `submitted`, `approved`, `rejected`
- `LastSyncedOn`

Add `WhatsAppMessageLog`:

- `NotificationDispatchId`
- `PhoneNumberMasked`
- `ExternalMessageId`
- `TemplateName`
- `Language`
- `Status`: `queued`, `sent`, `delivered`, `read`, `failed`
- `ErrorCode`
- `ErrorMessage`
- `PayloadJson`
- `CreatedOn`
- `UpdatedOn`

Add `CustomerCommunicationPreference`:

- `UserId`
- `Channel`
- `OptIn`
- `OptInSource`
- `OptInOn`
- `OptOutOn`

## Transactional Templates

Initial templates:

- Payment succeeded.
- Payment failed.
- Booking confirmed.
- Reminder 24 hours before service.
- Reminder 2 hours before service.
- Provider assigned/accepted.
- Provider on the way.
- Provider arrived.
- Service completed.
- Booking cancelled.
- Refund processed.

Each template needs:

- Arabic copy.
- English copy.
- Variable list.
- Fallback text for missing optional variables.
- Link/deep link rules.

## Sending Flow

1. Order event is created.
2. Notification service resolves customer language and opt-in.
3. Template is selected.
4. `NotificationDispatch` is created.
5. WhatsApp dispatcher sends template message.
6. External message ID is stored.
7. Status webhooks update message log.
8. Failures retry only when safe.

## Webhook Receiver Requirements

Add:

- `GET /api/v1/integrations/whatsapp/webhook` for Meta verification challenge if using direct Cloud API.
- `POST /api/v1/integrations/whatsapp/webhook` for statuses and inbound messages.

Requirements:

- Verify signature if provided by integration type.
- Store raw payload in `WebhookInbox`.
- Process asynchronously.
- Do not crash on unknown event type.

## Customer Experience

- WhatsApp messages should be concise and action-oriented.
- Include order number and service date/time.
- Do not include full address unless explicitly needed.
- Include support contact only when useful.
- Add unsubscribe/help handling for non-transactional future use.

## OTP Boundary

For now:

- WhatsApp is transactional only.
- OTP remains Firebase.

If WhatsApp OTP is later approved:

- Use authentication templates.
- Implement backend-owned OTP flow.
- Add rate limiting and anti-abuse.
- Keep it separate from order notification dispatch.

## Acceptance Criteria

- Approved Arabic and English templates exist for v1 events.
- Customer receives WhatsApp confirmation after payment success if opted in.
- Message status is stored and visible to support/admin.
- Failed sends are visible and retried according to policy.
- No marketing messages are sent through transactional flow.

## Risks

- Template approval can delay launch.
- Business Manager/phone number ownership issues can block direct Cloud API setup.
- WhatsApp policies distinguish utility/authentication/marketing categories.
- Customer opt-in and privacy policy must be reviewed.

## Owner Decisions

- Direct Cloud API or BSP?
- Use WhatsApp for all customers by default, or only customers who opt in?
- Should WhatsApp replace push for critical order updates, or be secondary?
- Is WhatsApp OTP desired later?

## Official References

- WhatsApp Cloud API: https://developers.facebook.com/docs/whatsapp/cloud-api
- WhatsApp message templates: https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-message-templates
- WhatsApp webhooks: https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks

