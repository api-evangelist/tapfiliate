---
generated: '2026-08-13'
method: generated
name: Recruit an affiliate and approve them for a program
description: >-
  Create an affiliate (or promote a prospect), place them in a group, add them to
  a program and approve them so their referral link starts earning.
api: openapi/tapfiliate-affiliates-api-openapi.yml
operations: [createAffiliate, listAffiliateGroups, setAffiliateGroup, addAffiliatToProgram, approveAffiliate, getAffiliations, getAffiliate]
source: >-
  operationIds verified verbatim in
  openapi/tapfiliate-affiliates-api-openapi.yml,
  openapi/tapfiliate-affiliate-groups-api-openapi.yml and
  openapi/tapfiliate-programs-api-openapi.yml; behaviour from
  https://tapfiliate.com/docs/rest/.
---

# Recruit an affiliate and approve them for a program

Approval is per **program**, not per affiliate. An affiliate can exist and still earn nothing until an affiliation is created and approved.

## Auth
- `X-Api-Key: <account key>` over HTTPS. See `authentication/tapfiliate-authentication.yml`.

## Steps

1. **Find the program** — `listPrograms` (`GET /programs/`) or `getProgram` (`GET /programs/{program_id}/`). Note `currency` and `cookie_time`.

2. **Create the affiliate** — `createAffiliate` (`POST /affiliates/`) with `email`, `firstname`, `lastname` and optional `meta_data`. The response carries the affiliate `id`, `referral_link` and `coupon`.
   - Already have a pending applicant? List them with `listAffiliateProspects` (`GET /affiliate-prospects/`) and read one with `getAffiliateProspect`.

3. **Optionally place them in a group** — `listAffiliateGroups` (`GET /affiliate-groups/`) to find the group id, then `setAffiliateGroup` (`PUT /affiliates/{affiliate_id}/group/`). Groups drive commission structure; create one first with `createAffiliateGroup` if needed.

4. **Add them to the program** — `addAffiliatToProgram` (`POST /programs/{program_id}/affiliates/`). This creates the affiliation.

5. **Approve them** — `approveAffiliate` (`PUT /programs/{program_id}/affiliates/{affiliate_id}/approved/`). Until this succeeds the affiliate is in the program but not earning. The inverse is `disapproveAffiliate` (`DELETE` on the same path).

6. **Verify** — `getAffiliations` (`GET /affiliates/{id}/programs/`) returns the affiliate's programs with `approved`, `coupon` and the per-affiliation `referral_link`.

## MLM
For sub-affiliate structures, set the parent with `setAffiliateParent` (`PUT /affiliates/{child_affiliate_id}/parent/`) and read the program's tiers with `listProgramMLMLevels` (`GET /programs/{program_id}/levels/`).

## Notes and gotchas
- Custom signup fields are readable with `getCustomFields` (`GET /affiliates/custom-fields/`) — check them before constructing `meta_data`.
- Attach internal context with `createAffiliateNote` (`POST /affiliates/{affiliate_id}/notes/`) or `setAffiliateMetaDataByKey`.
- No idempotency key exists (`conventions/tapfiliate-conventions.yml`). A retried `createAffiliate` can create a duplicate — search `listAffiliates` by email before retrying.
- Errors are a bare `{"message": "..."}`; a `422` on step 2 usually means a duplicate or malformed email. See `errors/tapfiliate-problem-types.yml`.
- Webhook triggers fire on "Affiliate created", "Affiliate added to program" and "Affiliate approved for program" — see `asyncapi/tapfiliate-webhooks.yml` if you want to react rather than poll.
