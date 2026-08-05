---
name: Build and attach a Madhive audience
description: Resolve segments into a boolean audience expression, de-duplicate it against existing audiences, create it, and attach it to a line item.
api: openapi/madhive-api-openapi-original.yml
operations:
  - getSegmentsByOrg
  - getAudiencesByOrg
  - searchAudiences
  - getAudienceById
  - createAudience
  - updateAudience
  - deleteAudienceById
  - updateLineItemById
  - createLineItem
---

# Build and attach a Madhive audience

A Madhive audience is a **boolean expression over segment and audience ids** (AND / OR /
NOT). The documented workflow searches before it creates, so you do not litter the account
with duplicate audiences that mean the same thing.

## Before you start

- Base URL `https://api2.madhive.com/api`, OAuth 2.0 bearer token on every call.
- No idempotency key exists — search before you create, always.

## Steps

1. **Enumerate the building blocks.**
   - `getSegmentsByOrg` — every segment available to the org, including retargeting
     segments. Supports `page_size` / `page_token` / `offset` pagination.
   - `getAudiencesByOrg` — existing custom audiences; pass `includeStdAuds` to fold in
     Madhive's standard audiences, `type` to filter, `search` for a text match.

2. **Compose the boolean expression** from those ids using AND / OR / NOT.

3. **De-duplicate first.** `searchAudiences` takes the boolean statement of segment and
   audience ids and tells you whether an audience with that logic already exists,
   returning its audience id if so. **Run this before every create.**

4. **Inspect what you found.** `getAudienceById` returns name, notes, the boolean logic,
   segment status, CPM and household count — enough to decide whether to reuse it.

5. **Create only if it did not exist.** `createAudience` with the same boolean statement.
   Returns `201` with the new audience id.

6. **Attach to a line item.**
   - Existing line item: `updateLineItemById` with the audience id.
   - New line item: pass the audience id on `createLineItem`.

7. **Editing.**
   - To change the audience **everywhere it is used**: read the boolean with
     `getAudienceById`, then `updateAudience` with the edited boolean. The change applies
     to every line item carrying that audience id.
   - To change it **for one line item only**: build a new audience (steps 2–5) and
     re-attach with `updateLineItemById`. Do not edit the shared audience.

8. **Retiring.** `deleteAudienceById` archives an audience — it is removed from anything
   it was attached to and from all future lists. There is no API unarchive.

## Error handling

Errors return `error` / `errors[]` / `status` / `transaction`; capture `transaction.id`.

- `400` — malformed boolean expression, or a segment/audience id that does not resolve.
- `401` — expired token; re-fetch and retry once.
- `429` — per-second tier limit. Segment and audience lists are the easiest place to trip
  this; paginate serially rather than in parallel.

## Gotchas

- `updateAudience` is global by design. If a caller says "just for this line item",
  that is a new audience, not an edit.
- Audience lists are paginated with the opaque `page_token` cursor; `offset` is a
  direction hint (1 next, -1 previous, 0 current), not a row index.
