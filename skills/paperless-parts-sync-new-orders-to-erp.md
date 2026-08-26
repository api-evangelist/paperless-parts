---
name: paperless-parts-sync-new-orders-to-erp
description: >-
  Pull newly placed Paperless Parts orders and everything attached to them so they can be written
  into an ERP, then write the ERP's own identifier back onto the order so the same order is never
  imported twice.
api: Paperless Parts API v2
version: '1.0'
generated: '2026-08-26'
method: generated
source: openapi/paperless-parts-v2-openapi.yml
operations:
  - NewOrderNumbers
  - OrderDetails
  - UpdateOrder
  - AccountDetails
  - ContactDetails
---

# Sync new Paperless Parts orders into an ERP

This is the single most common Paperless Parts integration: an order is placed in Paperless Parts,
and a job/work order has to appear in the shop's ERP.

## Before you start

- Base URL: `https://api.paperlessparts.com/v2`
- Every request carries `Authorization: API-Token <api_token>`. The token is generated in the
  Paperless Parts app under **Settings > Integrations > API Token** and is account-wide.
- There is **no idempotency header**. The watermark below is what keeps this safe.

## Steps

1. **Find what is new.** `GET /orders/public/new` (`NewOrderNumbers`) returns the order numbers
   placed since your last watermark. Persist the highest number you have processed; this endpoint
   plus your own watermark is the only replay protection the API offers.
2. **Pull each order in full.** `GET /orders/public/{orderNumber}` (`OrderDetails`) returns the
   whole order graph in one call — `order_items[]`, each item's `components[]` with `material`,
   `shop_operations[]`, `material_operations[]`, `supporting_files[]`, plus `contact`, `customer`,
   `billing_info`, `payment_details`, `salesperson` and `estimator`. You do not need to walk
   children separately; `GET` responses nest their associations.
3. **Resolve the customer if your ERP needs more than the nested snapshot.**
   `GET /accounts/public/{accountId}` (`AccountDetails`) and
   `GET /contacts/public/{contactId}` (`ContactDetails`). Both accept `erp_code` and
   `null_erp_code` filters on their list forms, which exist precisely so an ERP can find records by
   its own key.
4. **Create the work order in the ERP.** Out of scope for this API, but do it before step 5 so the
   identifier you write back is real.
5. **Write the ERP identifier back.** `PATCH /orders/public/{orderNumber}` (`UpdateOrder`) with the
   ERP code. This is what makes the sync convergent: a later pass can filter on `erp_code` /
   `null_erp_code` to see exactly which orders have not landed yet.

## Rules that matter here

- **Writes are PATCH, not PUT.** Omitted fields keep their current value. Sending `null` for a
  required field is an error, not a clear.
- **No retry safety.** If step 5 times out, re-read the order before re-sending — there is no
  `Idempotency-Key`.
- **`facilitate_order` has no reversal.** `POST /orders/public/facilitate_order` turns a quote into
  an order and no cancel, void or unfacilitate operation exists in either API version. Do not call
  it speculatively.
- **Errors are thin.** The spec declares only 404 (and one 400 in v1) with `text/plain` bodies. The
  live host actually returns `{"error", "detail", "path", "status_code"}` JSON. Parse defensively
  and treat any non-2xx as retryable-with-backoff only if the request was a GET.
- **No rate limits are published.** Pace your polling yourself; see
  `rate-limits/paperless-parts-rate-limits.yml`.
