# 08 - Salla Website Booking Integration Spec

Status: draft for review, no implementation included.

## Problem

Maujood has a website hosted on Salla that customers can book from, and wants it integrated with the backend. The backend currently supports mobile and portal orders, but no Salla channel, no Salla webhook receiver, no order mapping, and no reconciliation job.

## Current Behavior

- Mobile app books through Maujood backend.
- Portal can create manual orders.
- No Salla APIs or webhooks are present in the backend.
- Orders do not have `Channel` or `ExternalOrderId`.

## Key Architecture Decision

There are two viable models.

### Option A - Salla is checkout/source for website orders

Customer books and pays on Salla. Salla webhook creates or updates a Maujood order.

Pros:

- Uses existing Salla storefront/checkout.
- Faster if Salla booking fields can capture all required data.

Cons:

- Salla order model may not fit location, vehicle, provider, slot requirements.
- Maujood backend receives final order after Salla, which can limit availability validation.

### Option B - Maujood backend is booking source, Salla is front-end channel

Salla website embeds or links to a Maujood booking flow/API. Maujood creates the order and handles payment.

Pros:

- One source of truth.
- Better control over provider availability, location polygons, vehicle data, payment metadata, promo/packages.

Cons:

- More custom web work.
- May be harder inside Salla hosted environment.

Recommended decision: if Salla cannot collect provider, precise address/map, vehicle, and time slot reliably, use Option B.

## Goals

- Website bookings create valid Maujood backend orders.
- Orders include website/Salla channel attribution.
- Payment and order status stay reconciled.
- Customer notifications work for Salla-originated orders.

## Non-Goals

- Do not duplicate product/provider catalog manually in multiple places long-term.
- Do not treat Salla order IDs as Maujood order IDs.
- Do not let Salla webhook create duplicate orders on retries.

## Data Model Requirements

Add to `Order` or related table:

- `Channel`: `mobile_app`, `admin_portal`, `salla`
- `ExternalOrderId`
- `ExternalSource`
- `ExternalStatus`
- `ExternalPayloadJson`
- `CustomerChannelMetadataJson`

Add `ExternalOrderMapping`:

- `Source`: `salla`
- `ExternalOrderId`
- `OrderId`
- `ExternalCustomerId`
- `LastSyncedOn`
- `SyncStatus`

Add `WebhookInbox`:

- `Source`
- `EventType`
- `ExternalEventId`
- `Signature`
- `PayloadHash`
- `PayloadJson`
- `ProcessedOn`
- `Status`
- `Error`

## Salla Webhook Requirements

Subscribe to relevant events:

- `order.created`
- `order.updated`
- `order.status.updated`
- `order.cancelled`
- `order.refunded`
- Potentially payment/transaction events if available/needed.

Security:

- Verify Salla signature header/secret.
- Return success only after storing webhook inbox record.
- Process asynchronously.
- Use `ExternalOrderId` and event id/payload hash for idempotency.

Salla documentation notes webhooks use signatures/secrets and retries if the endpoint does not return success.

## Order Mapping Requirements

Map Salla fields to Maujood:

- Customer name.
- Customer phone number.
- Service/product selected.
- Provider selected or eligible provider selection.
- Booking date.
- Booking time slot.
- Address text.
- Latitude/longitude if available.
- Vehicle make/model/year/color/plate if available.
- Payment status.
- Coupon/discount, if Salla handled it.

If Salla cannot provide structured fields, require a custom booking form/widget before allowing production use.

## Availability Requirements

Before creating Maujood order from Salla:

- Validate product/service exists.
- Validate provider is active.
- Validate provider serves location polygon.
- Validate selected slot is still available.
- Validate user/customer exists or create customer.

If invalid:

- Store failed webhook processing.
- Alert operations.
- Do not silently create a broken order.

## Payment Strategy

If Salla handles payment:

- Store Salla payment reference.
- Mark order paid only after trusted Salla paid/final status.
- Do not run Moyasar payment flow for that order.

If Maujood handles payment:

- Website should call Maujood place order and Moyasar payment flow.
- Salla order may be informational only, or not used for checkout.

## API Requirements

Backend:

- `POST /api/v1/integrations/salla/webhook`
- `GET /api/v1/integrations/salla/reconciliation`
- Internal service `SallaOrderMapper`
- Admin view of failed Salla events.

Optional website booking API:

- Public service/catalog endpoints scoped for web.
- Slot preview endpoint.
- Place order endpoint with anti-abuse/rate limits.

## Reconciliation

Add scheduled reconciliation:

- Fetch recent Salla orders via API.
- Compare with `ExternalOrderMapping`.
- Flag missing or mismatched orders.
- Alert operations for manual fix.

## Acceptance Criteria

- Salla order webhook creates one Maujood order, not duplicates.
- Invalid/missing booking fields route to operations queue.
- Salla cancellation/refund updates Maujood order state according to policy.
- Maujood portal can filter orders by `Channel = salla`.
- Customer receives the same confirmation notifications as mobile customers.

## Risks

- Salla checkout may not support precise booking metadata without customization.
- Webhooks can arrive out of order or be retried.
- Payment source of truth must be agreed before implementation.
- Catalog mismatch between Salla and Maujood can create wrong services/prices.

## Owner Decisions

- Should Salla take payment, or should Maujood/Moyasar take payment?
- Can Salla product pages collect map location and vehicle details cleanly?
- Should Salla website use the same provider/time-slot availability as the app?
- Who owns catalog updates: Salla, Maujood admin, or both?

## Official References

- Salla authorization: https://docs.salla.dev/421118m0
- Salla webhooks: https://docs.salla.dev/421119m0
- Salla API overview: https://docs.salla.dev/

