---
name: paperless-parts-sync-customers-and-pricing-tables
description: >-
  Keep Paperless Parts accounts, contacts, facilities and billing addresses in step with an
  external CRM/ERP, and bulk-load the custom tables and purchased components that pricing logic
  reads from.
api: Paperless Parts API v2
version: '1.0'
generated: '2026-08-26'
method: generated
source: openapi/paperless-parts-v2-openapi.yml
operations:
  - ListAccounts
  - CreateAccoumt
  - AccountDetails
  - UpdateCompany
  - ListContacts
  - CreateContact
  - UpdateCustomer
  - CreateFacility
  - CreateBillingAddress
  - ListPaymentTerms
  - GetCustomTablesList
  - GetCustomTableDetails
  - BulkUpsertCustomTable
  - ListPurchasedComponents
  - PostPurchasedComponentCSV
---

# Sync customers and pricing reference data

Two jobs share one skill because they share one discipline: both are bulk, both are convergent, and
both write into data the shop's pricing depends on.

Base URL `https://api.paperlessparts.com/v2`, header `Authorization: API-Token <api_token>`.

## Customers

The shape is: an **Account** (a company) has zero or more **Contacts** (people, unique by email),
plus **Facilities** (ship-to) and **BillingAddresses** (bill-to).

1. **Reconcile by your own key first.** `GET /accounts/public` (`ListAccounts`) and
   `GET /contacts/public` (`ListContacts`) both accept `erp_code` and `null_erp_code`. Use
   `erp_code` to find what you already own and `null_erp_code=true` to find records created in the
   Paperless Parts UI that your system has never seen. `search` and `account_id` also filter.
2. **Create what is missing.** `POST /accounts/public` (`CreateAccoumt` — the misspelling is in the
   provider's spec) and `POST /contacts/public` (`CreateContact`). Write associations flat:
   `account_id`, not a nested account.
3. **Update what drifted.** `PATCH /accounts/public/{accountId}` (`UpdateCompany`) and
   `PATCH /contacts/public/{contactId}` (`UpdateCustomer`).
4. **Attach addresses.** `POST /accounts/public/{accountId}/facilities` (`CreateFacility`) and
   `POST /accounts/public/{accountId}/billing_addresses` (`CreateBillingAddress`).
5. **Payment terms** are their own resource: `GET`/`POST /customers/public/payment_terms`.

## Custom tables and purchased components

Custom tables are the lookup tables P3L pricing reads. Purchased components are bought-in parts
with their own column schema.

1. **Discover the schema before writing.** `GET /suppliers/public/custom_tables`
   (`GetCustomTablesList`) then `GET /suppliers/public/custom_tables/{tableName}`
   (`GetCustomTableDetails`) — the response carries the column `config` and the `related_entities`
   that depend on the table.
2. **Bulk load.** `PUT /suppliers/public/custom_tables/{tableName}/row/bulk`
   (`BulkUpsertCustomTable`).
3. **Purchased components.** `GET`/`POST /suppliers/public/purchased_components`, the column
   surface at `/suppliers/public/purchased_component_columns`, and CSV round-tripping via
   `GET`/`POST /suppliers/public/purchased_components_csv`.

## Rules that matter here

- **The bulk writes have no undo.** `BulkUpsertCustomTable` and `PostPurchasedComponentCSV` replace
  rows in tables that live pricing depends on, and no snapshot, rollback or restore operation
  exists. Read the current table with `GetCustomTableDetails` and keep the response before you
  overwrite — that is the only rollback you get.
- **Check `related_entities`** on a custom table before changing its columns; other pricing objects
  reference it.
- **Contact and account deletes are v2-only.** `DeleteContact` and `DeleteAccount` do not exist in
  v1, so a record created through v1 can only be removed through v2.
- **Paginate.** `page` and `page_size` on every list; there is no cursor.
- **No idempotency, no rate limits published.** Sequence bulk loads, do not parallelise them, and
  back off on any non-2xx.
