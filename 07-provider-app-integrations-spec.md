# 07 - Provider App Integrations Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants to integrate with provider applications. Today the backend stores providers, provider schedules, polygons, and products, but providers do not have external API credentials, webhook endpoints, app users, assignment states, or a contract for accepting/rejecting/updating orders.

## Current Behavior

- Providers are configured in portal endpoints.
- Provider availability is stored as weekly durations and time gaps.
- Provider service areas are stored as polygons.
- Orders reference `ProviderId`.
- Operations/admin can change order status manually.
- There is no provider-facing authentication or order API.

## Goals

- Allow provider applications to receive new orders.
- Allow providers to accept/reject/update order progress.
- Support future ETA/live notification updates.
- Keep Maujood backend as the source of truth for customer-visible order state.
- Make every external provider callback idempotent and auditable.

## Non-Goals

- Do not let provider systems directly update payment records.
- Do not expose customer PII beyond what is needed to deliver service.
- Do not require all providers to integrate at once.

## Integration Models

### Model A - Provider Pull API

Provider app polls Maujood API.

Pros:

- Easier for providers.
- Simpler security and retry handling.

Cons:

- Less real-time.
- More API traffic.

### Model B - Maujood Webhooks to Provider

Maujood sends order webhooks to provider endpoint.

Pros:

- Real-time.
- Better for operations automation.

Cons:

- Requires provider to build webhook receiver.
- Requires retry, signature, failure handling.

Recommended v1: support pull API first and add webhooks for providers that can support them.

## Data Model Requirements

Add `ProviderIntegration`:

- `ProviderId`
- `IntegrationType`: `api_key`, `webhook`, `oauth`
- `ApiKeyHash`
- `WebhookUrl`
- `WebhookSecret`
- `AllowedIpRanges`
- `IsActive`
- `CreatedOn`
- `LastUsedOn`

Add `ProviderOrderAssignment`:

- `OrderId`
- `ProviderId`
- `Status`: `pending`, `sent`, `accepted`, `rejected`, `expired`, `cancelled`
- `ProviderExternalOrderId`
- `SentOn`
- `AcceptedOn`
- `RejectedOn`
- `RejectionReason`

Add `ProviderEventLog`:

- `ProviderId`
- `OrderId`
- `EventType`
- `ExternalEventId`
- `PayloadJson`
- `Status`
- `CreatedOn`

## Provider API Requirements

Provider authentication:

- API key in header for v1.
- Store only hashed API keys.
- Rotate keys from portal.
- Optionally restrict IP per provider.

Endpoints:

- `GET /api/v1/provider/orders`
  - List assigned orders with filters.

- `GET /api/v1/provider/orders/{id}`
  - Detail needed to fulfill service.

- `POST /api/v1/provider/orders/{id}/accept`
  - Provider accepts.

- `POST /api/v1/provider/orders/{id}/reject`
  - Provider rejects with reason.

- `POST /api/v1/provider/orders/{id}/status`
  - Update progress: `en_route`, `arrived`, `started`, `completed`.

- `POST /api/v1/provider/orders/{id}/eta`
  - Optional ETA update.

Use `Idempotency-Key` header for all mutating requests.

## Maujood-to-Provider Webhook Requirements

Webhook events:

- `provider_order.created`
- `provider_order.cancelled`
- `provider_order.updated`

Security:

- HMAC signature header, e.g. `X-Maujood-Signature`.
- Timestamp header.
- Reject stale timestamps.
- Retry with exponential backoff.
- Deactivate webhook after repeated failures and alert operations.

Payload:

- Maujood order id.
- External id if assigned.
- Service name.
- Scheduled time.
- General address/location needed for service.
- Customer contact only if operationally approved.
- Vehicle information needed for service.
- Price/payment status summary, not raw payment method details.

## Order State Rules

- Provider can accept/reject only assigned pending orders.
- Provider cannot complete unpaid order.
- Provider cannot cancel customer order directly; rejection routes to operations or reassignment.
- Provider completion can mark order as service completed only if operations policy allows.
- All provider updates create `OrderProgressEvent`.

## Portal Requirements

- Configure integration type and credentials per provider.
- View provider assignment state.
- Resend order to provider.
- Manually override assignment after audit reason.
- View provider event logs.

## Acceptance Criteria

- Provider can list assigned orders using secure API key.
- Provider can accept/reject an order exactly once.
- Provider status update changes Maujood order progress and triggers customer notification.
- Duplicate provider requests do not duplicate state changes.
- Operations can inspect provider API/webhook failures.

## Risks

- Providers may have inconsistent capabilities; contract must be stable but optional fields need graceful handling.
- Customer privacy needs careful review before exposing phone/address details.
- Idempotency is mandatory or provider retries will corrupt state.
- Current order status model is too limited and should be expanded before deep provider integration.

## Owner Decisions

- Do providers see customer phone number directly?
- Can providers mark service completed, or only operations?
- Should providers receive orders before or after payment success?
- What SLA should trigger escalation if provider does not accept?

