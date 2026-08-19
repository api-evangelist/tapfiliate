---
generated: '2026-08-13'
method: generated
name: Approve commissions and pay affiliates
description: >-
  Review pending commissions, approve the good ones, read the resulting balances
  and settle them with a payment. The money-moving flow — treat it accordingly.
api: openapi/tapfiliate-payments-api-openapi.yml
operations: [listCommissions, getCommission, approveCommission, disapproveCommission, listAllBalances, getAffiliateBalances, listPayoutMethods, createPayment, listPayments, cancelPayment]
source: >-
  operationIds verified verbatim in
  openapi/tapfiliate-commissions-api-openapi.yml,
  openapi/tapfiliate-balances-api-openapi.yml,
  openapi/tapfiliate-payments-api-openapi.yml and
  openapi/tapfiliate-affiliates-api-openapi.yml.
---

# Approve commissions and pay affiliates

## Auth
- `X-Api-Key: <account key>` over HTTPS. Tapfiliate's own documentation warns that this key **can approve commissions** — it is the highest-privilege credential in the product and it has no scoping. See `authentication/tapfiliate-authentication.yml`.

## Steps

1. **List pending commissions** — `listCommissions` (`GET /commissions/`). Page with `?page` (1-based, 25 per page); follow the `Link` header `rel="next"`. There is no total count, so walk until `next` is absent.

2. **Inspect one** — `getCommission` (`GET /commissions/{commission_id}/`) returns `amount`, `approved`, `commission_type`, `affiliate` and `conversion_id`.

3. **Approve or reject** —
   - `approveCommission` (`PUT /commissions/{commission_id}/approved/`)
   - `disapproveCommission` (`DELETE /commissions/{commission_id}/approved/`)
   - To correct an amount before approving, `updateCommission` (`PATCH /commissions/{commission_id}/`).

4. **Read balances** — `listAllBalances` (`GET /balances/`) across the account, or `getAffiliateBalances` (`GET /affiliates/{affiliate_id}/balances/`) for one affiliate. Balances are per **currency**; an affiliate can hold several.

5. **Check they can be paid** — `listPayoutMethods` (`GET /affiliates/{affiliate_id}/payout-methods/`). An affiliate with no payout method has nowhere for the money to go.

6. **Settle** — `createPayment` (`POST /payments/`). Confirm with `listAffiliatePayments` (`GET /affiliates/{affiliate_id}/payments/`) or `listPayments` (`GET /payments/`); `getPayment` returns `amount`, `currency`, `paid_at` and `status`.

7. **Reverse if needed** — `cancelPayment` (`DELETE /payments/{id}/`).

## Retry safety — this is the dangerous one

**There is no idempotency key on this API.** `createPayment` has no `external_id` equivalent and no uniqueness constraint. A network timeout on step 6 is indistinguishable from a failure, and a blind retry can pay an affiliate twice.

Do this instead:
1. Before any retry of `createPayment`, call `listAffiliatePayments` (or `listPayments`) and check whether a payment matching the amount, currency and window already exists.
2. Only retry when you have confirmed it does not.
3. If a duplicate lands, `cancelPayment` is the remedy — but treat that as incident recovery, not flow control.

See `conventions/tapfiliate-conventions.yml`.

## Errors
- `401` — bad or missing key.
- `404` — unknown commission, affiliate or payment id.
- `422` — validation failure on `createPayment` or `updateCommission`; a single `message` string, no field pointer.
- No `429` or `5xx` is declared in the contract. Read `X-Ratelimit-Remaining` and back off before it reaches zero (`rate-limits/tapfiliate-rate-limits.yml`).
See `errors/tapfiliate-problem-types.yml`.

## Notes
- No test mode exists (`sandbox/tapfiliate-sandbox.yml`). Every call in this skill moves real money in the production account.
- The Tapfiliate MCP server **cannot** perform any step here — it is read-only and the provider states explicitly that it cannot create payments (`mcp/tapfiliate-mcp.yml`). An agent that reports on payouts must still hand the write to a REST client.
- Webhook triggers fire on "Commission created", "Commission updated" and "Payment created" (`asyncapi/tapfiliate-webhooks.yml`).
