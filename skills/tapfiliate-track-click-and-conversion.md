---
generated: '2026-08-13'
method: generated
name: Track a click and a conversion over REST
description: >-
  Record an affiliate referral click, then attribute a purchase to it, using only
  the REST API — the integration path Tapfiliate documents for merchants who do
  not want to run the browser tracker.
api: openapi/tapfiliate-clicks-api-openapi.yml
operations: [createClick, createConversion, getConversion]
source: >-
  Grounded in https://tapfiliate.com/docs/integrations/rest-api/. operationIds
  verified verbatim in openapi/tapfiliate-clicks-api-openapi.yml and
  openapi/tapfiliate-conversions-api-openapi.yml.
---

# Track a click and a conversion over REST

The REST-only attribution path. Two calls, in order, with a value carried between them.

## Auth
- `X-Api-Key: <account key>` on every request, over HTTPS. Plain HTTP fails. See `authentication/tapfiliate-authentication.yml`.
- One static account key, no scoping — the same key that can approve commissions and create payments. Never put it in client-side code.

## Base
- `https://api.tapfiliate.com/1.6`

## Steps

1. **Record the click** — `createClick` (`POST /clicks/`).
   On any page visit, check the URL for a `?ref=` query parameter. If present, send its value as `referral_code`.
   The response is a JSON object with an `id` — the **click id**. Persist it in a cookie whose lifetime is at least the program's `cookie_time` (read it with `getProgram`).

2. **Attribute the purchase** — `createConversion` (`POST /conversions/`).
   Send `click_id` (from step 1), `external_id` and `amount`.
   - `external_id` is yours to generate — an order number or transaction id. Tapfiliate requires it to be **unique for each conversion**.
   - `amount` is the total value; percentage commissions are calculated from it.

3. **Confirm** — `getConversion` (`GET /conversions/{conversion_id}/`) returns the conversion with its `affiliate` and embedded `commissions`.

## Retry safety — read this before writing the client

There is **no idempotency key** on this API (`conventions/tapfiliate-conventions.yml`). A retried `createConversion` can create a second conversion.

- Generate `external_id` **before the first attempt** and reuse the identical value on every retry. The uniqueness constraint on `external_id` is the only replay guard Tapfiliate provides.
- `createClick` has **no** such guard. A retried click POST creates a second click. Treat a click POST as fire-once and fall back to re-reading rather than re-posting.

## Errors
- `422` — validation failed, including a duplicate `external_id`. The body is `{"message": "..."}` with no field pointer, so parse the prose. See `errors/tapfiliate-problem-types.yml`.
- `401` — missing or invalid `X-Api-Key`.
- `404` — the click id or conversion id does not exist.
- Rate limiting is signalled by `X-Ratelimit-Remaining`; the exhaustion status code is not published (`rate-limits/tapfiliate-rate-limits.yml`).

## Notes
- For SaaS, subscription or lead-gen, track a **Customer** instead at step 2 so recurring and lifetime commissions can attach later — see `tapfiliate-track-customer-for-recurring-commissions.md`.
- There is no test mode. Every call here writes to the production account (`sandbox/tapfiliate-sandbox.yml`).
