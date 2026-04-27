# 11 - Moyasar Account Update Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants to update the Moyasar account. The app and backend currently depend on Moyasar publishable/secret keys and webhook secret. Changing the account affects mobile remote config, backend config, webhooks, Apple Pay, STC Pay, and reconciliation.

## Current Behavior

- App reads `MOYASAR_API_KEY` from Firebase Remote Config.
- App sends payment metadata `order_id`.
- Backend validates Moyasar webhook `secret_token` against `MoyaserSecretKey`/`MoyasarSecretKey` config naming.
- Current webhook endpoint documented/used is `/api/v1/Webhook/webhook4`.
- Webhook code expects event names matching Moyasar docs, including `payment_paid` and `payment_faild`.
- Current failure path in code should be reviewed because failed payment path records success status in existing source.

## Goals

- Move payment processing to the correct Moyasar account with minimal downtime.
- Verify card, Apple Pay, and STC Pay behavior in staging before production cutover.
- Keep payment webhook secure and idempotent.
- Avoid customers paying in old account while backend expects new account.

## Non-Goals

- Do not change pricing or product catalog.
- Do not implement new payment methods beyond account migration.
- Do not rotate unrelated secrets in this spec, except where needed for payment safety.

## Required Moyasar Setup

In Moyasar dashboard:

- Confirm account/business/KYC status.
- Obtain production publishable API key.
- Obtain production secret API key.
- Configure webhook endpoint URL.
- Configure webhook secret token.
- Enable desired events:
  - `payment_paid`
  - `payment_faild` or current Moyasar spelling shown in dashboard/docs
  - `payment_refunded`
  - `payment_voided`
- Configure Apple Pay certificate/merchant settings.
- Confirm STC Pay activation if needed.
- Confirm settlement bank account.
- Confirm test/sandbox keys separately from production keys.

## Apple Pay Requirements

- Merchant ID in code/config must match Apple Developer and Moyasar dashboard.
- Apple Pay capability must be enabled in Xcode.
- Payment processing certificate must be active.
- Test on physical iOS device.
- Confirm Apple Pay button visibility and payment completion.

Current iOS code uses merchant identifier `merchant.com.maujood.app`; verify whether this remains correct.

## Config Changes

Backend:

- `MoyasarSecretKey` or `MoyaserSecretKey` naming should be standardized.
- Store secret key and webhook token in secure environment/secret store.
- Do not hardcode in `appsettings.json` for production.

App:

- Firebase Remote Config:
  - `MOYASAR_API_KEY`
  - `IS_APPLE_PAY_ENABLED`
  - `IS_STC_PAY_ENABLED`
- Verify QA/prod environment separation.

## Cutover Plan

Phase 1 - Preparation:

- Add idempotency to webhook payment transaction creation.
- Add webhook inbox logs.
- Fix payment failure status mapping.
- Add payment account/environment labels to logs.

Phase 2 - Staging:

- Configure staging Moyasar keys.
- Run test payments for card, STC Pay, Apple Pay.
- Verify webhook receives and records correct transaction.
- Verify order details and notifications.

Phase 3 - Production cutover:

- Schedule low-traffic window.
- Update backend secrets.
- Update Moyasar dashboard webhook URL/secret.
- Update Firebase Remote Config publishable key.
- Force refresh Remote Config or wait known fetch interval.
- Monitor payment success/failure for first day.

Phase 4 - Cleanup:

- Disable old account webhook or keep temporarily for reconciliation only.
- Reconcile payments between old and new account during transition window.
- Document final keys ownership in local/secret manager inventory.

## Webhook Idempotency Requirements

Before cutover, ensure:

- Payment ID is unique in `OrderTransaction` or `PaymentEvent`.
- Duplicate webhook for same payment does not create duplicate transaction.
- Failed payment creates failed status, not success.
- Refund/void events are handled or stored for manual review.
- Unknown event type is stored and acknowledged or rejected based on policy.

## Monitoring Requirements

Track:

- Payment attempts.
- Payment completed in SDK.
- Payment webhook received.
- Payment transaction created.
- Payment mismatch: SDK says paid but webhook missing after threshold.
- Webhook unauthorized attempts.
- Moyasar API errors.

## Acceptance Criteria

- New Moyasar publishable key is active in app remote config.
- Backend accepts webhooks only with correct new secret.
- Test payments create exactly one success transaction.
- Failed payments create failed transaction/status.
- Apple Pay and STC Pay are either verified active or intentionally disabled.
- Operations can reconcile old/new account orders during transition.

## Risks

- Remote Config caching can cause some app clients to keep old publishable key temporarily.
- Apple Pay may fail if merchant ID/certificate/account mismatch.
- Webhook secret mismatch can make paid orders stay pending.
- Existing webhook failure-status bug can hide failed payments if not fixed before migration.

## Owner Decisions

- Exact production cutover date/time.
- Keep old Moyasar account active temporarily or disable immediately?
- Are Apple Pay and STC Pay required on day one after account update?
- Who owns Moyasar dashboard/admin access going forward?

## Official References

- Moyasar docs: https://docs.moyasar.com/
- Moyasar webhooks: https://docs.moyasar.com/guides/dashboard/setting-up-webhooks/
- Moyasar create payment: https://docs.moyasar.com/api/payments/01-create-payment/
- Moyasar Apple Pay iOS: https://docs.moyasar.com/sdk/ios/apple-pay-payments-integration/

