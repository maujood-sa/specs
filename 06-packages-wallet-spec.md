# 06 - Packages and Wallet Credit Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants packages such as a number of services or credit. There is currently no wallet, package, entitlement, subscription, or redemption model in the backend. Orders are one-off paid bookings through Moyasar.

## Product Options

Three package models are possible:

1. Service bundles:
   - Customer buys a fixed number of services.
   - Example: 5 car washes for 300 SAR.

2. Wallet credit:
   - Customer buys credit balance.
   - Example: pay 500 SAR, receive 550 SAR credit.

3. Membership/subscription:
   - Customer pays recurring fee for benefits.
   - Example: monthly package with discounted services.

Recommended v1: service bundles or wallet credit, not recurring membership. Recurring billing adds payment, cancellation, compliance, and renewal complexity.

## Goals

- Sell packages safely through Moyasar.
- Allow package balance/entitlement redemption against orders.
- Show customers remaining balance/services.
- Give admin visibility and adjustment controls.
- Preserve an audit trail for every balance movement.

## Non-Goals

- Do not implement recurring subscriptions in v1.
- Do not allow negative balances.
- Do not make packages transferable between customers unless explicitly approved.

## Data Model Requirements

Add `PackageProduct`:

- `Id`
- `NameAr`
- `NameEn`
- `DescriptionAr`
- `DescriptionEn`
- `PackageType`: `service_bundle`, `wallet_credit`
- `Price`
- `Currency`
- `CreditAmount`
- `ServiceCount`
- `EligibleProductIds`
- `EligibleProviderIds`
- `StartsOn`
- `EndsOn`
- `ExpiresAfterDays`
- `IsActive`

Add `CustomerPackage`:

- `Id`
- `UserId`
- `PackageProductId`
- `Status`: `pending_payment`, `active`, `expired`, `cancelled`, `refunded`
- `PurchasedOn`
- `ExpiresOn`
- `RemainingCredit`
- `RemainingServices`
- `OriginalCredit`
- `OriginalServices`
- `OrderTransactionId`

Add `PackageLedgerEntry`:

- `CustomerPackageId`
- `UserId`
- `OrderId`
- `EntryType`: `purchase`, `redeem`, `refund`, `manual_adjustment`, `expiry`
- `CreditDelta`
- `ServiceDelta`
- `BalanceAfterCredit`
- `BalanceAfterServices`
- `Reason`
- `CreatedBy`
- `CreatedOn`

## API Requirements

Mobile:

- `GET /api/v1/packages`
- `GET /api/v1/my-packages`
- `POST /api/v1/packages/{id}/purchase`
- `POST /api/v1/orders/price-preview`
- `POST /api/v1/orders/place-order` with optional package redemption.

Portal:

- Package CRUD.
- Customer package lookup.
- Manual adjustment with required reason.
- Package purchase/redemption export.

## Purchase Flow

1. Customer views package.
2. Backend creates package purchase record as `pending_payment`.
3. App pays with Moyasar, with metadata `package_purchase_id`.
4. Moyasar webhook confirms payment.
5. Backend activates `CustomerPackage` and creates ledger entry.
6. Customer receives package purchase notification.

## Redemption Flow

1. Customer selects service/provider/time.
2. Backend price preview returns eligible packages.
3. Customer chooses package redemption.
4. Backend validates package:
   - active
   - not expired
   - enough balance/services
   - eligible for product/provider
5. Backend places order and creates ledger redemption entry in same transaction.
6. Total due is reduced to zero or partial amount.

## Refund and Cancellation Rules

Need owner policy before implementation:

- If unused package is refunded, set package `refunded` and ledger reversal.
- If partially used package is refunded, calculate refundable amount manually or by formula.
- If order paid by package is cancelled before service, restore service count/credit.
- If service completed, do not restore unless admin override.

## UX Requirements

- Packages tab in profile or home.
- Package cards show remaining credit/services, expiry, eligible service types.
- Checkout price breakdown shows package applied.
- Order details show package redemption.

## Accounting Requirements

- Every balance change must be ledgered.
- Manual adjustments require admin identity and reason.
- Paid package revenue and redeemed service value should be reportable separately.
- Package liability balance should be exportable for finance.

## Acceptance Criteria

- Customer can purchase package and see it active after Moyasar payment.
- Customer can redeem package against eligible order.
- Ledger entries reconcile to remaining balance.
- Expired/cancelled/refunded packages cannot be redeemed.
- Admin can audit every manual adjustment.

## Risks

- Packages affect revenue recognition and refund policy.
- Partial package payments plus promocodes can become complex quickly.
- Without idempotent payment webhook handling, package purchase can activate twice.
- Current order pricing model needs cleanup before package redemption.

## Owner Decisions

- Service bundles or wallet credit for v1?
- Should packages expire?
- Can packages be used with promocodes?
- Can packages be provider-specific?
- How should partially used refunds be handled?

