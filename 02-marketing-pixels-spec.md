# 02 - Marketing Pixels Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants marketing pixels and conversion tracking. The current system has PostHog hooks and an incomplete Amplitude compatibility layer, but it does not have a governed marketing event taxonomy, server-side conversion outbox, web pixel integration, or deduplication strategy across app, website, Salla, and backend orders.

## Current Behavior

- App reads `POSTHOG_API_KEY`, `POSTHOG_HOST`, and `AMPLITUDE_KEY` from Firebase Remote Config.
- PostHog is wired on Android and iOS shells.
- Amplitude setup is currently partial in the compatibility layer.
- Backend does not send marketing events.
- Salla website booking integration is not implemented yet.
- Purchase/payment truth currently lives in backend `OrderTransaction` rows created by Moyasar webhook.

## Goals

- Track the booking funnel from discovery to paid order.
- Support Google Analytics/Google Ads and Meta Pixel/Conversions API.
- Avoid duplicate purchase counting across app, web, backend, and Salla.
- Make backend payment success the source of truth for purchase conversion.

## Non-Goals

- Do not implement broad customer data platform features yet.
- Do not send sensitive personal data unless explicitly approved and hashed according to platform requirements.
- Do not rely only on browser pixels for purchase attribution.

## Event Taxonomy

Use a shared event naming map across app, website, backend, and marketing platforms.

Core funnel:

- `app_open`
- `view_home`
- `select_location`
- `view_service_list`
- `view_service`
- `select_provider`
- `view_provider`
- `select_vehicle`
- `select_schedule_slot`
- `begin_checkout`
- `payment_method_selected`
- `purchase`
- `payment_failed`
- `order_cancelled`
- `order_completed`
- `rating_requested`
- `promo_applied`
- `package_purchased`
- `package_redeemed`

Recommended marketing mappings:

- GA4 `view_item_list` for service/provider lists.
- GA4 `view_item` for product/provider detail.
- GA4 `begin_checkout` when user reaches payment method selection.
- GA4 `purchase` only after backend payment success.
- Meta `ViewContent`, `InitiateCheckout`, `Purchase`, and later `Subscribe` or custom events for packages.

## Data Model Requirements

Add backend-side tracking support:

- `MarketingEvent`
  - `Id`
  - `EventName`
  - `Source`: `mobile_app`, `web`, `salla`, `backend`
  - `UserId`
  - `AnonymousId`
  - `OrderId`
  - `TransactionId`
  - `ProviderId`
  - `ProductId`
  - `Value`
  - `Currency`
  - `CouponCode`
  - `EventTime`
  - `PayloadJson`
  - `ConsentState`
  - `CreatedOn`

- `MarketingDispatch`
  - `MarketingEventId`
  - `Destination`: `ga4`, `google_ads`, `meta_capi`, `posthog`, `amplitude`
  - `ExternalEventId`
  - `Status`
  - `AttemptCount`
  - `LastError`
  - `LastAttemptOn`

Use an outbox pattern so payment webhooks can commit order/payment state before dispatching marketing calls.

## Deduplication Strategy

- Use backend `Order.Id` as `transaction_id` for GA4 purchase.
- Use a stable `event_id` for Meta Pixel + Conversions API deduplication, e.g. `purchase:{orderId}:{paymentId}`.
- For web/Salla frontend pixels, emit non-final checkout events client-side.
- Emit purchase from backend only after Moyasar confirms payment success.
- If frontend must also emit purchase, it must use the same event ID and transaction ID.

## Consent and Privacy

- Add consent fields to app/web before marketing tags go beyond strictly necessary analytics.
- Document what data is sent to each platform.
- Hash customer identifiers where required.
- Do not send address text, full phone numbers, plate numbers, or raw Firebase tokens to marketing platforms.
- Support opt-out/limited tracking if required by App Store, Play, or local privacy policy commitments.

## Website/Salla Requirements

- Install Google tag/GA4 and Meta Pixel on Salla if Salla permits custom scripts/pixels.
- Fire web events for service views and checkout starts.
- Send paid purchase conversion from backend after Salla order maps to Maujood order.
- Preserve UTM and click IDs from website into backend order metadata:
  - `utm_source`
  - `utm_medium`
  - `utm_campaign`
  - `gclid`
  - `gbraid`
  - `wbraid`
  - `fbclid`
  - Meta `_fbp`/`_fbc` if available and approved.

## App Requirements

- Add a common analytics abstraction in shared KMM code.
- Log event once from ViewModel/business milestone, not from every composable render.
- Include platform and app version.
- Keep PostHog as product analytics if desired.
- Add Meta App Events or server-side Conversions API for app conversions only after consent decision.

## Backend Requirements

- Add `/api/v1/analytics/events` for app/web non-final events if server-side event relay is approved.
- Emit backend events on:
  - order placed
  - payment paid
  - payment failed
  - order completed
  - refund
  - promo/package purchase/redeem later
- Add admin config for active marketing destinations and credentials.

## Acceptance Criteria

- A paid order creates one and only one purchase conversion in backend dispatch.
- GA4 DebugView shows expected checkout/purchase events in staging.
- Meta Events Manager receives browser and/or server events with dedupe IDs.
- Marketing events never contain raw OTPs, raw Firebase tokens, full phone numbers, or vehicle plate numbers.
- A source/channel field can distinguish mobile app, Salla, and admin-created orders.

## Risks

- Browser-only pixels can be blocked, duplicated, or lost.
- Salla may restrict custom checkout scripts depending on plan/setup.
- Cross-domain/Salla attribution may require careful UTM and cookie preservation.
- App Store privacy labels may need updates if new SDKs are installed.

## Owner Decisions

- Which platforms are required at launch: GA4, Google Ads, Meta, TikTok/Snapchat later?
- Are we allowed to use customer phone/email for advanced matching if hashed?
- Should server-side purchase events be the only source of truth? Recommended: yes.

## Official References

- GA4 ecommerce events: https://developers.google.com/analytics/devguides/collection/ga4/ecommerce
- Meta Pixel: https://developers.facebook.com/docs/meta-pixel/get-started
- Meta Conversions API: https://developers.facebook.com/docs/marketing-api/conversions-api

