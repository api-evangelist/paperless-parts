---
name: paperless-parts-build-a-quote
description: >-
  Compose a Paperless Parts quote from parts up — create the quote, add line items, attach
  operations, add-ons and discounts, upload supporting files, then move it through its status.
api: Paperless Parts API v2
version: '1.0'
generated: '2026-08-26'
method: generated
source: openapi/paperless-parts-v2-openapi.yml
operations:
  - CreateGeometricPart
  - CreateManualPart
  - GetPartStatus
  - CreateQuote
  - CreateQuoteItem
  - updateQuoteItem
  - CreateOperation
  - bulkUpdateOperations
  - CreateDiscount
  - createQuoteFile
  - SetQuoteStatus
  - QuoteDetails
---

# Build a Paperless Parts quote through the API

Quoting is where Paperless Parts' data model has the most depth: a **Quote** has
**QuoteItems**, each of which has **QuoteComponents** (the part tree), each with a **material**,
**material_operations**, **shop_operations**, **add-ons**, **discounts** and **pricing_items** —
and every operation carries **costing_variables**, which is where P3L pricing logic reads from.

Base URL `https://api.paperlessparts.com/v2`, header `Authorization: API-Token <api_token>`.

## Steps

1. **Get a part into the library.**
   - CAD-driven: `POST /parts/public/geometric_part` (`CreateGeometricPart`). Geometry analysis is
     asynchronous — poll `GET /parts/public/{partUuid}/status` (`GetPartStatus`) until it is ready.
     Do not attach a part to a quote before its status says it has been processed.
   - No CAD: `POST /parts/public/manual_part` (`CreateManualPart`), or
     `POST /parts/public/manual_assembly` (`CreateManualAssembly`) for a BOM.
2. **Create the quote.** `POST /quotes/public` (`CreateQuote`). Associations are written flat, by
   id — `contact_id`, not a nested contact object. Reading it back nests them.
3. **Add line items.** `POST /quotes/public/items` (`CreateQuoteItem`), referencing the part by
   `part_uuid`. Amend with `PATCH /quotes/public/items/{quoteItemUuid}` (`updateQuoteItem`).
4. **Shape the pricing.**
   - `POST /quotes/public/operations` (`CreateOperation`) then
     `PATCH /quotes/public/operations/{operationUuid}` (`updateOperation`), or amend many at once
     with `PATCH /quotes/public/items/{quoteItemUuid}/bulk_update_operations`
     (`bulkUpdateOperations`).
   - Valid operation and add-on shapes come from the shop's own configuration — read them first
     with `GET /processes/public/operation_definitions` (`OperationDefintionList`),
     `GET /processes/public/add_on_definitions` (`AddOnDefinitionList`),
     `GET /processes/public/discount_definitions` (`DiscountDefinitionList`),
     `GET /processes/public/processes` (`ProcessesList`) and
     `GET /processes/public/materials` (`ListMaterials`). Do not invent definition ids.
   - Discounts: `POST /quotes/public/discounts` (`CreateDiscount`).
   - Per-quantity pricing: `GET`/`PATCH /quotes/public/pricing_item_quantities/{uuid}`.
5. **Attach files.** `POST /quotes/public/{quoteUuid}/files` (`createQuoteFile`).
6. **Move it.** `PATCH /quotes/public/{quoteNumber}/status_change` (`SetQuoteStatus`).
7. **Verify.** `GET /quotes/public/{quoteNumber}` (`QuoteDetails`) returns the whole nested graph.

## Rules that matter here

- **Read definitions before writing pricing.** Operations, add-ons, discounts and materials are all
  configured per shop. Guessing an id is the most common failure mode here.
- **Every composition step is reversible, but no window is published.** `deleteQuoteItem`,
  `deleteOperation`, `deleteDiscount` and `deleteQuoteFile` all exist, and `SetQuoteStatus` can be
  re-issued with the prior status. Paperless Parts does not state how long a delete can be undone or
  whether it is soft — see `conventions/paperless-parts-conventions.yml`.
- **Parts cannot be deleted.** No part delete operation is published in either version.
- **`facilitate_order` is the point of no return.** `POST /orders/public/facilitate_order` converts
  the quote to an order and has no reversal in the API.
- **PATCH is partial.** Omitting a field leaves it alone; sending `null` for a required field errors.
