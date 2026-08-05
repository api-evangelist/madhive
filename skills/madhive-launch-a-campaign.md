---
name: Launch a Madhive campaign
description: Create an advertiser, a campaign, a line item and a creative flight in the Madhive DSP, then set the campaign live.
api: openapi/madhive-api-openapi-original.yml
operations:
  - createAdvertiser
  - createAgency
  - createCampaign
  - getProductsByOrg
  - getMetros
  - createLineItem
  - createCreative
  - createCreativeLineItem
  - activateCampaignById
  - getCampaignById
---

# Launch a Madhive campaign

Madhive's campaign object is a container. Its dates are **derived** from its line items,
and nothing delivers until you activate it. Build bottom-up, activate last.

## Before you start

- Base URL: `https://api2.madhive.com/api`
- Get a token: `POST https://api2.madhive.com/oauth/token` with
  `grant_type=client_credentials`, your `client_id` and `client_secret`.
  Send it as `Authorization: Bearer <jwt>` on every call.
- All times are **UTC**, formatted `yyyy:mm:ddThh:mm:ss`.
- **There is no idempotency key.** A create call that times out may still have
  succeeded — re-read with the matching list operation before retrying, never blind-retry.
- Rate limit is per second and depends on the account tier (Basic 2 → Premium 15).
  Serialize these steps rather than fanning out.

## Steps

1. **Confirm the org.** `getOrgs` returns the organizations the credentials can act on.
   Everything below is scoped to one org.

2. **Create or find the advertiser.** `getAdvertisers` first — advertisers are reused
   across campaigns. If it does not exist, `createAdvertiser` with a unique name, a
   domain, an IAB category code, an optional agency and an optional `externalId`.
   Keep the returned advertiser id.
   - Need an agency first? `getAgencies`, then `createAgency` (name + optional external id).
   - The IAB category set on the advertiser can be reused on all of its creatives.

3. **Create the campaign.** `createCampaign` with the campaign name, the advertiser id,
   and optionally an agency or stations. Keep the returned campaign id.
   - Madhive seeds default dates (start = next day 00:00, end = start + 3 years).
     Do not treat those as your flight — they collapse to the real line-item span
     once a line item exists.

4. **Pick targeting inputs.**
   - `getProductsByOrg` — a product is a preset for publisher group, device
     distribution, frequency cap and dayparting. Cost+ products accept a max eCPM on
     the line item; Rate Card products are fixed price.
   - `getMetros` — metro/DMA codes. `getStations` for call letters/DMA if you are
     targeting broadcast.
   - `getPublisherGroupsByOrg` and `getAudiencesByOrg` for an existing publisher group
     and audience, or build them with the companion skills.
   - `getContextualSegments` for contextual targeting and avoidance.

5. **Create the line item.** `createLineItem` with the campaign id, name, start and end
   dates, the product id, **either** an impression goal **or** a budget goal (not both —
   this cannot be switched after go-live), pace period (day or lifetime), a required
   daily frequency cap, device distribution summing to at least 100%, a country, and any
   audience / publisher group / geo / daypart targeting.
   - Start date cannot be in the past. End date cannot precede the start date.

6. **Upload the creative.** `createCreative` with name, advertiser id, IAB category code,
   creative type (Tag or CDN), the VAST tag or CDN URL, and a click-through. Media files
   such as MP4 cannot be uploaded through the API — tags and CDN URLs only.
   The response returns validation attributes; they are advisory and do not block upload.
   - Optional: `getCreativesMediaTrustStatus` to batch-check the malware-scan status.

7. **Schedule the creative on the line item.** `createCreativeLineItem` with the creative
   id, the line item id, flight dates **inside** the line item's start/end window,
   a relative weight, and any third-party tracking pixels.

8. **Activate.** `activateCampaignById` sets the campaign live along with all of its line
   items and creatives. To go live on one line item only, POST the line-item activation
   path instead (`/v1/campaigns/{cid}/lineitems/{id}/activation`).
   - If the start time has already passed, Madhive rewrites it to now + 10 minutes.
   - Every required field must be present or activation fails.

9. **Verify.** `getCampaignById` (with `includeLineItems=true`) and
   `getLineItemsByCampaignId` return the current status of the campaign and its line items.

## Error handling

Errors return a JSON envelope — `error`, `errors[]`, `status`, `transaction` — not
RFC 9457 problem details. Log `transaction.id`; it is the trace id Madhive support needs.

- `400` — missing or invalid parameters. The `errors[]` array names the offending fields.
- `401` — token expired or invalid. Re-fetch from the token endpoint and retry once.
- `403` — the credentials lack permission for that org or resource.
- `409` — conflict, e.g. deleting an optimization template still bound to active line items.
- `429` — you exceeded the per-second tier limit. Back off; no `Retry-After` is published.
- `500` / `503` — Madhive-side. Check https://madhive.checkly-status-page.com/ before retrying.

## Gotchas

- Campaign start/end are computed from the earliest and latest line-item dates.
- Extending a line item's end date moves it to **Paused** — it must be set live again,
  and creative flights are **not** extended automatically.
- After go-live you can still edit most line-item attributes, but not the start date once
  it has passed, not the pace period, and not the pacing strategy (impressions ↔ budget).
- There is no unarchive operation in the API; unarchiving is UI-only.
