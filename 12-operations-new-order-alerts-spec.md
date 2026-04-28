# 12 - Operations New Order Alerts Spec

Status: draft for review, backend configuration foundation partially implemented.

## Problem

Operations needs to be notified of every new order. Today operations can view portal order listings, but the backend does not send dedicated staff alerts when orders are created, paid, unassigned, provider-rejected, or approaching service time.

## Current Behavior

- Mobile order creation writes order and pending transaction.
- Moyasar webhook records payment result.
- Moyasar webhook handling now has a webhook inbox and duplicate event/transaction guard, which is a prerequisite for avoiding duplicate operations alerts later.
- Portal can list all orders.
- Backend now has `OperationsAlertRule` and `OperationsAlertLog` foundation tables plus admin APIs to configure rules/recipients.
- No email/WhatsApp/Slack/Teams/ops push alert dispatcher exists yet.

## Implementation Status - 2026-04-28

Implemented:

- Prerequisite webhook idempotency foundation for payment events.
- First backend CSV order export endpoint, which can help operations review orders manually while alerting is not built.
- Added backend operations alert rule/log models, EF migration, and admin configuration endpoints.

Still pending:

- No automatic operations email/WhatsApp alert is implemented yet.
- Email provider/sender setup and actual dispatch worker are still pending.
- Needs order-event/outbox wiring before reliable alert delivery.

## Goals

- Notify operations when a new order needs attention.
- Reduce missed paid bookings.
- Escalate orders that remain unaccepted/unassigned.
- Keep alerts auditable and configurable.

## Non-Goals

- Do not build full workforce management.
- Do not send customer-facing messages from this feature.
- Do not expose operations-only notes to customers.

## Alert Events

Initial v1:

- `ops.order.created_pending_payment`
- `ops.order.payment_succeeded`
- `ops.order.payment_failed`
- `ops.order.cancelled`
- `ops.order.provider_rejected`
- `ops.order.unaccepted_after_threshold`
- `ops.order.service_starting_soon`

Recommended first production alert:

- Payment succeeded/new paid order.

This avoids noisy alerts for abandoned/pending payment orders.

## Channels

Phase 1:

- Email to operations group mailbox.

Phase 2:

- WhatsApp to operations group/approved recipients.
- Slack/Teams if the business uses it.

Phase 3:

- Admin portal notification center.
- Escalation to manager if unaccepted after X minutes.

## Data Model Requirements

Implemented foundation `OperationsAlertRule`:

- `EventKey`
- `Channel`
- `Recipients`
- `IsActive`
- `TriggerDelayMinutes`
- `CooldownMinutes`
- `TemplateKey`

Implemented foundation `OperationsAlertLog`:

- `Id`
- `OperationsAlertRuleId`
- `EventKey`
- `OrderId`
- `PayloadJson`
- `Status`
- `CreatedOn`
- `Channel`
- `Recipient`
- `LastError`
- `SentOn`

## Email Content

Subject examples:

- `New paid order #{OrderId} - {ServiceDate} {ServiceTime}`
- `Provider rejected order #{OrderId}`
- `Order #{OrderId} starts in 2 hours`

Body should include:

- Order ID.
- Customer name and phone.
- Service/product/variant.
- Provider.
- Service date/time in Riyadh timezone.
- Address summary/link to maps if available.
- Vehicle summary.
- Payment status/amount.
- Link to portal order details.

## WhatsApp Content

Use operations-approved template if sent through WhatsApp Business.

Keep message concise:

- Order ID.
- Service and time.
- Provider.
- Customer phone.
- Portal link.

Do not use customer-facing templates for operations alerts.

## Scheduling and Escalation

Escalation examples:

- Paid order not accepted/confirmed within 10 minutes.
- Service begins in 2 hours and order is not confirmed.
- Provider rejects order.

Rules must re-check current order state before sending.

## API Requirements

Portal/admin:

- Implemented: `GET /api/v1/Configuration/operations-alert-rules`
- Implemented: `POST /api/v1/Configuration/operations-alert-rules`
- Implemented: `PUT /api/v1/Configuration/operations-alert-rules/{id}`

Future:

- `GET /api/v1/operations/alerts`
- `PUT /api/v1/operations/alerts/{id}/acknowledge`

Internal:

- `OperationsAlertService.Enqueue(eventKey, orderId, payload)`
- Alert dispatcher using outbox/background worker.

## Acceptance Criteria

- New paid order sends one operations email.
- Duplicate payment webhook does not send duplicate alert.
- Operations can see sent/failed alerts in backend/portal.
- Alert includes enough details to act without opening multiple systems.
- Escalation rule can be configured and disabled.

## Risks

- Alert fatigue if pending-payment orders are included.
- Email deliverability requires a proper provider/domain setup.
- WhatsApp group messaging through official API has constraints; direct group alerts may require a different pattern or BSP support.
- Without an admin frontend repo, portal UI work may be blocked, but backend can still send email alerts.

## Owner Decisions

- Which channel first: email, WhatsApp, or both?
- Who are the recipients?
- Should alerts trigger on order created or only payment succeeded?
- What is the provider acceptance SLA before escalation?
