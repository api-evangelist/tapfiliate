---
generated: '2026-08-13'
method: generated
name: Track a customer for recurring and lifetime commissions
description: >-
  Bind a signup to the referring click so future renewals keep paying the same
  affiliate — the SaaS/subscription path, distinct from one-off conversion
  tracking.
api: openapi/tapfiliate-customers-api-openapi.yml
operations: [createClick, createCustomer, createConversion, getCustomer, cancelCustomer]
source: >-
  Grounded in https://tapfiliate.com/docs/integrations/rest-api/ and
  https://tapfiliate.com/docs/guides/how-to-setup-recurring-or-lifetime-commissions-using-the-rest-api/.
  operationIds verified verbatim in
  openapi/tapfiliate-customers-api-openapi.yml,
  openapi/tapfiliate-clicks-api-openapi.yml and
  openapi/tapfiliate-conversions-api-openapi.yml.
---

# Track a customer for recurring and lifetime commissions

For SaaS, subscription and lead-gen, the durable attribution object is a **Customer**, not a Conversion. The customer holds the link back to the originating click, so every later renewal can be credited to the same affiliate.

## Auth
- `X-Api-Key: <account key>` over HTTPS. See `authentication/tapfiliate-authentication.yml`.

## Prerequisite
Lifetime/recurring commissions must be toggled **on** for the program in the Tapfiliate dashboard. The API will happily accept the calls below with it off, and no recurring commission will ever be generated.

## Steps

1. **Record the click** — `createClick` (`POST /clicks/`) with `referral_code` from `?ref=`. Keep the returned click `id`.

2. **Create the customer at signup** — `createCustomer` (`POST /customers/`) with `click_id` and your own `customer_id`. Use an id that is stable for the lifetime of the account — a user id, account number or email.

3. **Record each billing event** — `createConversion` (`POST /conversions/`) with the `customer_id` and a fresh unique `external_id` per invoice, plus `amount`. The affiliate credit follows the customer.

4. **Read back** — `getCustomer` (`GET /customers/{id}/`) returns the customer with its `status`, originating `click` and `meta_data`.

5. **On churn** — `cancelCustomer` (`PUT /customers/{id}/status/`) marks the customer cancelled so recurring commissions stop.

## Metadata
Attach your own context with `replaceCustomerMetaData` (`PUT /customers/{id}/meta-data/`) or per key with `setCustomerMetaDataByKey` (`PUT /customers/{id}/meta-data/{key}/`). Metadata is an arbitrary key/value map; there is no schema. See `conventions/tapfiliate-conventions.yml`.

## Retry safety
No idempotency key exists. Reuse a pre-generated `external_id` on every retry of step 3 — it is the only uniqueness constraint the API enforces. Step 2 has no such guard; check with `getCustomer` before re-posting.

## Errors
- `422` — validation failure, including a duplicate `external_id`. Single `message` string, no field pointer.
- `404` — unknown click id, customer id or conversion id.
- `401` — bad or missing key.
See `errors/tapfiliate-problem-types.yml`.

## Notes
- Migrating an older integration that attributes by `referral_code` instead of `click_id`? Tapfiliate documents the upgrade at /docs/guides/migrating-from-referral-code-to-click-id-based-api-tracking/. Click-id attribution is the form this skill uses.
- Entity relationships: `data-model/tapfiliate-data-model.yml`.
