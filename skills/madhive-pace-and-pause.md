---
name: Pace, pause and reconcile Madhive delivery
description: Adjust impression and budget goals, reschedule flights, pause or resume campaigns, line items and individual creatives, and archive cleanly.
api: openapi/madhive-api-openapi-original.yml
operations:
  - getCampaignsByOrg
  - getCampaignById
  - getLineItemsByCampaignId
  - getLineItemById
  - updateLineItemImpression
  - updateLineItemBudget
  - updateLineItemSchedule
  - updateLineItemById
  - activateCampaignById
  - deactivateCampaignById
  - deactivateLineItemById
  - activateCreativeById
  - deleteCreativeLineItem
  - deleteCampaignById
  - deleteLineItemById
  - deleteCreativeById
---

# Pace, pause and reconcile Madhive delivery

In-flight changes. Use the narrow PATCH operations where they exist — they avoid
resending the whole line item and avoid clobbering fields you did not intend to touch.

## Before you start

- Base URL `https://api2.madhive.com/api`, OAuth 2.0 bearer token on every call.
- All times UTC. No idempotency key — read state back rather than blind-retrying.

## Read current state

- `getCampaignsByOrg` — campaigns and ids for the org.
- `getCampaignById` with `includeLineItems=true` — campaign plus its line items.
- `getLineItemsByCampaignId` / `getLineItemById` — line item status and detail.
- `getCreativesByCampaignId` — creative flights on a campaign.

## Adjust goals without a full update

- `updateLineItemImpression` — `PATCH /v1/lineitems/{id}/impression-goal`.
- `updateLineItemBudget` — `PATCH /v1/lineitems/{id}/budget-goal`.
- `updateLineItemSchedule` — `PATCH /v1/lineitems/{id}/schedule`.

These are the endpoints introduced in the February 2025 change period specifically so
callers do not have to resend every line-item field. Prefer them over
`updateLineItemById` for goal and schedule changes.

**You cannot switch pacing strategy.** A line item pacing to impressions cannot be moved
to budget pacing (or back) after it is live, and the pace period (day vs lifetime) is
likewise fixed.

## Pause and resume

- Campaign: `deactivateCampaignById` pauses the campaign and every line item under it;
  `activateCampaignById` resumes.
- Line item: `deactivateLineItemById` pauses one line item and all of its creatives;
  POST `/v1/campaigns/{cid}/lineitems/{id}/activation` sets it live again.
- Single creative flight: `deleteCreativeLineItem` pauses one creative on one line item;
  `activateCreativeById` resumes it.

## Extend a flight

`updateLineItemById` can move the end date at any time, including into the past-dated
case, to extend the line item. Two consequences to handle:

1. Extending sets the line item to **Paused** — it must be set live again.
2. Creative flights are **not** extended automatically. Update each creative schedule
   with `updateCreativeLineItem` or the flight will end before the line item does.

The end date can never be moved before the start date.

## Editing a completed campaign

Completed campaigns and line items are still editable. On a campaign: name, advertiser,
agency. On a line item: name and end date.

## Archive

`deleteCampaignById`, `deleteLineItemById`, `deleteCreativeById`,
`deleteAudienceById`, `deleteAdvertiserById`, `deleteAgencyById`,
`deletePublisherGroupById` and `deleteRetargetingSegmentById` all **archive** rather than
hard-delete. An archived object is removed from anything it was attached to and from
future lists. **There is no unarchive operation in the API** — that is UI-only, so
confirm before archiving on a user's behalf.

## Error handling

- `400` — an edit the lifecycle forbids (past start date, pacing-strategy switch,
  end before start).
- `401` — expired token; re-fetch and retry once.
- `429` — per-second tier limit (Basic 2 → Premium 15). Poll status serially.
- `500` / `503` — check https://madhive.checkly-status-page.com/ before retrying.

Log `transaction.id` from every error envelope.
