# 05 - Promocodes Spec

Status: draft for review, no implementation included.

## Problem

Maujood wants discount codes. The backend currently has product `Price` and `Discount`, but no promocode, redemption, eligibility, stacking, or audit model. Existing `Discount` appears to be used as final payable amount in places, not discount amount, so pricing needs to be normalized before adding codes.

## Current Behavior

- Product has `Price` and `Discount`.
- Variant has `VariantPrice` and `VariantDiscount`.
- `PlaceOrder` sets order `Price` and `Discount` from product/variant.
- Payment amount is `order.Discount` when present, otherwise `order.Price`.
- No coupon input exists in app order flow.
- No admin coupon endpoints exist.

## Goals

- Allow customers to apply valid discount codes at checkout.
- Support owner/admin creation and reporting.
- Prevent overuse, fraud, and accidental revenue leakage.
- Preserve exact pricing snapshot on the order.

## Non-Goals

- Do not build a broad loyalty engine in the first version.
- Do not allow arbitrary stacking without explicit approval.
- Do not recalculate historical order totals after promo rules change.

## Pricing Terms

Before implementation, standardize:

- `Subtotal`: base service price before discounts.
- `ProductDiscount`: built-in product sale discount, if any.
- `PromoDiscount`: discount produced by promo code.
- `PackageCreditApplied`: amount or entitlement used from package/wallet.
- `TotalDue`: amount to charge through Moyasar.

Current `Order.Discount` should be treated carefully because it appears to mean discounted price/final amount in existing code.

## Promo Types

Initial supported types:

- Percentage discount, e.g. 20%.
- Fixed amount discount, e.g. 25 SAR.
- First-order-only discount.
- Provider/product/child-service scoped discount.

Later:

- Free add-on.
- Minimum-spend offer.
- Referral code.
- Campaign-specific codes.

## Data Model Requirements

Add `PromoCode`:

- `Id`
- `CodeNormalized`
- `DisplayCode`
- `NameAr`
- `NameEn`
- `DescriptionAr`
- `DescriptionEn`
- `DiscountType`: `percentage`, `fixed_amount`
- `DiscountValue`
- `MaxDiscountAmount`
- `MinOrderAmount`
- `Currency`
- `StartsOn`
- `EndsOn`
- `TotalUsageLimit`
- `PerUserUsageLimit`
- `FirstOrderOnly`
- `IsActive`
- `CreatedBy`

Add `PromoScope`:

- `PromoCodeId`
- `ScopeType`: `provider`, `product`, `child_service`, `user`, `channel`
- `ScopeId` or `ScopeValue`

Add `PromoRedemption`:

- `PromoCodeId`
- `UserId`
- `OrderId`
- `CodeSnapshot`
- `SubtotalSnapshot`
- `DiscountAmount`
- `TotalAfterDiscount`
- `RedeemedOn`
- `Status`: `reserved`, `confirmed`, `released`, `voided`

## API Requirements

Mobile:

- `POST /api/v1/promocodes/validate`
  - Inputs: code, product, provider, variant, address, user, subtotal, channel.
  - Output: validity, localized message, discount amount, final total.

- `POST /api/v1/Order/place-order`
  - Add optional `PromoCode`.
  - Server validates again and snapshots discount.

Portal:

- Create/update/deactivate promo.
- List promos with filters.
- View redemptions.
- Export promo usage.

## Validation Rules

Promo is valid only if:

- Active.
- Current Riyadh time is within date range.
- Usage limits not exceeded.
- User-specific limits not exceeded.
- Product/provider/service/channel scope matches.
- Minimum order amount met.
- First-order-only rule passes, if configured.
- Code is not combined with incompatible package/discount rules.

Use database transaction or reservation to prevent race conditions on usage limits.

## Checkout UX

- Promo field appears on payment summary.
- Apply button validates server-side.
- Show discount line item in price breakdown.
- Allow removing the code before payment.
- If code expires or becomes invalid before place order/payment, show clear error and recalculate.

## Payment/Webhook Behavior

- Promo discount should be applied before creating Moyasar payment request.
- Store original subtotal and discount amount on order.
- Webhook should not recalculate promo.
- Refunds should reference paid total and promo snapshot.

## Analytics

Track:

- `promo_code_entered`
- `promo_code_valid`
- `promo_code_invalid`
- `promo_applied`
- `promo_removed`
- `purchase` with `coupon` parameter for GA4/Meta mapping.

## Acceptance Criteria

- Valid code changes total due correctly.
- Invalid code returns localized reason.
- Same user cannot exceed per-user limit.
- Total usage limit is safe under concurrent orders.
- Promo amount is visible in app order details and portal order listing.
- Webhook/payment amount matches backend order total.

## Risks

- Existing naming of `Discount` may cause double-discount or undercharge if not cleaned up first.
- Race conditions can overspend limited campaigns.
- Manual admin orders need the same validation or explicit override audit.
- Refund rules for promo orders need financial approval.

## Owner Decisions

- Can promos stack with product discounts?
- Can promos stack with packages/wallet credit?
- Should admins be able to override invalid promo rules manually?
- Do we need influencer/referral reporting in v1?

