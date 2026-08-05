---
name: Build a Madhive publisher group
description: Pull the publisher inventory list, assign distribution caps that sum to at least 100%, create the group, and apply it to a line item.
api: openapi/madhive-api-openapi-original.yml
operations:
  - getPublishersByOrg
  - getPublisherGroupsByOrg
  - getPublisherGroupById
  - createPublisherGroup
  - updatePublisherGroupById
  - deletePublisherGroupById
  - updateLineItemById
  - getOptimizationTemplates
---

# Build a Madhive publisher group

A publisher group is a curated allow-list of publishers a line item may deliver against —
NBC, Hulu, ESPN, Samsung TV Plus, Tubi and so on — each carrying a percentage
distribution cap.

## Before you start

- Base URL `https://api2.madhive.com/api`, OAuth 2.0 bearer token on every call.
- The caps **must sum to at least 100%**. If the group sums to less, the line item
  mathematically cannot deliver in full, and Madhive blocks the upload.

## Steps

1. **Get the inventory.** `getPublishersByOrg` returns the full publisher list available
   to the org. Filter with `mediaType` where relevant. Paginate with
   `page_size` / `page_token`.

2. **Check for an existing group.** `getPublisherGroupsByOrg` lists the org's groups;
   `getPublisherGroupById` returns the publishers inside one.

3. **Assign caps.** Pick the publishers to target and give each a percentage distribution
   cap. Validate the sum ≥ 100 before you send.

4. **Create.** `createPublisherGroup` with the name and the publisher/cap list.
   Keep the returned group id.

5. **Apply to a line item.** Pass the publisher group id on `createLineItem` for a new
   line item, or `updateLineItemById` for an existing one.

6. **Edit.** Read the current membership with `getPublisherGroupById`, make the changes,
   then `updatePublisherGroupById`.
   - Edits propagate to line items automatically, but the delivery system refetches line
     item details only about **every 30 minutes**. To make an edit take effect
     immediately, re-apply the group id with `updateLineItemById`.

7. **Retire.** `deletePublisherGroupById` archives the group; archived groups disappear
   from future lists. No API unarchive exists.

## Related: optimization templates (Supply Guardrails)

`getOptimizationTemplates`, `getOptimizationTemplateById`, `createOptimizationTemplate`,
`updateOptimizationTemplate` and `deleteOptimizationTemplate` manage Supply Guardrails —
publisher settings, publisher caps, bundle rules and channel rules applied as a reusable
template. `deleteOptimizationTemplate` returns **409** when the template is still assigned
to one or more active line items; detach it first.

## Error handling

- `400` — caps summing below 100%, an unknown `pubId`, or an invalid `mediaType`.
- `401` — expired token.
- `409` — optimization template still bound to active line items.
- `429` — per-second tier limit; the publisher list is large, so paginate serially.

Every error body carries `transaction.id` — log it.
