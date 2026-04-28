# 09 - Admin Order Export Spec

Status: draft for review, phase 1 backend CSV partially implemented.

## Problem

Operations wants to export orders from the admin portal. The backend has an `all-orders-portal` listing endpoint, but no export endpoint, no CSV/XLSX generation, no async export jobs, and no export audit log.

## Current Behavior

- `GET /api/v1/Order/all-orders-portal` returns paginated orders.
- Filters are limited by current query parameters.
- No admin frontend source was found locally, only backend portal APIs.
- A backend CSV endpoint now exists at both `GET /api/v1/Order/export-orders-csv` and `GET /api/v1/Order/export`.
- The endpoint requires `OrderControllerAccess`, returns UTF-8 CSV with BOM, and supports query filters.
- Synchronous export date range is configurable with `Exports:Orders:MaxDateRangeDays`.
- The current implementation does not yet add audit logs, async jobs, or role-based field masking.

## Implementation Status - 2026-04-28

Implemented:

- Backend synchronous CSV export endpoint: `GET /api/v1/Order/export-orders-csv`.
- Route alias: `GET /api/v1/Order/export`.
- Query filters: created date range, service date range, order status, provider, product, user, and keyword search.
- Configurable synchronous range limit through `Exports:Orders:MaxDateRangeDays`, defaulting to 31 days.
- Export columns include order, customer, provider, service/product, transaction, address, and vehicle details.
- CSV uses UTF-8 BOM for Arabic/Excel compatibility.

Still pending before this is production-complete:

- Add export audit log: requester, filters, row count, download timestamp.
- Add async export job for large exports.
- Confirm whether plate number, address, and customer phone should remain visible for every portal role.
- Wire the admin portal frontend to call the new endpoint.

## Goals

- Allow authorized portal users to export order data.
- Preserve the same filters as the portal listing.
- Support finance/operations columns needed for follow-up.
- Prevent large exports from timing out or leaking data.

## Non-Goals

- Do not build a full BI dashboard.
- Do not expose exports to mobile users.
- Do not export secrets, tokens, or raw webhook payloads.

## Export Formats

Phase 1:

- CSV UTF-8 with BOM for Arabic/Excel compatibility.

Phase 2:

- XLSX with formatted columns.
- Async export jobs for large date ranges.

## Required Filters

Match or extend order listing:

- Date range by order created date.
- Service date range.
- Order status mapped and raw status.
- Payment status.
- Provider.
- Product/service.
- Customer phone/name keyword.
- Channel: mobile, portal, Salla later.
- Promo code later.
- Package redemption later.

## Export Columns

Core:

- Maujood order ID.
- Channel.
- External order ID, if any.
- Created date/time in Riyadh timezone.
- Service date.
- Service time.
- Raw order status.
- Mapped order status.
- Payment status.
- Payment method.
- Transaction reference/RRN.
- Currency.
- Original price.
- Product discount/final product amount.
- Promo discount, when added.
- Package amount/services redeemed, when added.
- Total paid.

Customer:

- Customer ID.
- Customer name.
- Customer mobile.
- Customer email.

Service:

- Parent/child service.
- Product name.
- Variant.
- Provider ID.
- Provider name.

Address/vehicle:

- Address title.
- Address details.
- Latitude.
- Longitude.
- Vehicle brand.
- Vehicle model.
- Vehicle year.
- Vehicle color.
- Plate number, if approved for export.

Audit:

- Last modified date.
- Completed/cancelled by, when available.

## API Requirements

Phase 1 synchronous:

- Current endpoints: `GET /api/v1/Order/export-orders-csv` and `GET /api/v1/Order/export`
- Query filters: `CreatedFrom`, `CreatedTo`, `ServiceDateFrom`, `ServiceDateTo`, `OrderStatusId`, `ProviderId`, `ProductId`, `UserId`, `SearchText`.
- `format=csv`
- Requires `OrderControllerAccess` or more specific `OrderExportAccess`.
- Returns file stream.

Phase 2 async:

- `POST /api/v1/exports/orders`
- `GET /api/v1/exports/{id}`
- `GET /api/v1/exports/{id}/download`

Add `ExportJob`:

- `Id`
- `Type`
- `RequestedBy`
- `FiltersJson`
- `Format`
- `Status`
- `FilePath`
- `RowCount`
- `Error`
- `CreatedOn`
- `CompletedOn`
- `ExpiresOn`

## Security Requirements

- Enforce portal auth and export permission.
- Log requester, filters, row count, and download timestamp.
- Limit maximum date range for synchronous exports.
- Mask or omit sensitive fields based on role.
- Expire generated files.

## Performance Requirements

- Use read-only DB context.
- Select only required columns.
- Stream CSV where possible.
- Avoid loading images/blobs.
- For large exports, require async job.

## Portal UX Requirements

Since frontend source was not found locally, backend spec should be delivered first.

Expected portal controls:

- Export button on orders page.
- Uses current filters.
- Shows progress for async export.
- Shows last generated export and expiration time.

## Acceptance Criteria

- Authorized admin can export filtered orders as CSV.
- Export row count matches portal filter count.
- Arabic text opens correctly in Excel.
- Large exports do not time out; they switch to async or return a clear error.
- Every export is auditable by admin user and filters.

## Risks

- Current listing mapping logic is embedded in service projection and may need refactor to avoid duplicated export logic.
- Plate/address exports may be sensitive.
- XLSX generation libraries may increase deployment footprint.
- Without frontend source, only backend export can be completed until portal app is available.

## Owner Decisions

- CSV only for v1, or XLSX required immediately?
- Should plate/customer phone/address stay available to `admin,operation`, or should field masking be added?
- Current synchronous export range is 31 days by config; confirm if this should change.
