# Research: Zernio coverage for Meta Ads and Google Ads

Ticket: danyelangel/rita#2. Researched 2026-10-14 from Zernio's official docs (docs.zernio.com) and marketing pages (zernio.com). Marketing pages are promotional; where docs and marketing disagree, docs win. "UNCONFIRMED" marks anything the docs did not state.

## Verdict

Zernio covers the MVP's Performance Marketing needs for both Meta Ads and Google Ads: reads (campaign tree, metrics, 730-day range), writes (create/edit/pause campaigns, budgets, audiences, customer lists) and conversion events (Meta Conversions API, Google offline/enhanced conversions via Data Manager API). The main risks are commercial and operational (ad pricing not fully published, the Facebook Page requirement, vendor dependency, a shared Google developer token), not missing endpoints.

## Coverage matrix against MVP needs

| MVP need | Meta Ads | Google Ads | Source |
|---|---|---|---|
| Diagnosis read: campaigns / ad sets / ads | Yes. Synced tree (`GET /v1/ads/tree`), list endpoints, and live read of campaigns + ad sets (`GET /v1/ads/accounts/live`, Meta and TikTok only) | Yes via synced tree (all campaign types incl. Performance Max) and raw GAQL passthrough. No live bulk read (501) | [get-ad-tree](https://docs.zernio.com/ad-campaigns/get-ad-tree.mdx), [live entities](https://docs.zernio.com/ad-accounts/get-ad-account-live-entities.mdx), [Google Ads](https://docs.zernio.com/platforms/google-ads) |
| Diagnosis read: spend, results | Yes: spend, impressions, reach, clicks, CTR/CPC/CPM, conversions, purchaseValue, ROAS, actions; breakdowns by age, gender, country, placement, device, region, creative asset | Summary metrics yes (spend, clicks, conversions, ROAS, impression share, Quality Score, search terms). Reach reported as 0, `actions` empty, no breakdowns on the analytics endpoint | [get-ad-analytics](https://docs.zernio.com/ad-insights/get-ad-analytics.mdx), [zernio.com/google-ads](https://zernio.com/google-ads) |
| History depth | Default 90 days, max 730 days per query. 202 with `backfillPending` until historical backfill completes | Same endpoints and limits. How far back GAQL goes is not stated (UNCONFIRMED) | [get-ad-analytics](https://docs.zernio.com/ad-insights/get-ad-analytics.mdx) |
| Ad review status | Yes (`reviewStatus`: in_review / approved / rejected / with_issues) | Google recommendations list/apply/dismiss; a generic ad review-verdict endpoint exists | [Meta ads](https://docs.zernio.com/platforms/meta-ads), [endpoint catalogue](https://docs.zernio.com/llms.txt) |
| Plan execution: create campaigns | Yes: campaign/ad set/ad in one call or level by level, 11 objectives, CBO/ABO, boost post, dry-run `validateOnly` | Yes: Search, Display, Performance Max, Demand Gen. Video views and conversion goals rejected at creation (422); Display needs two images; PMax is created paused | [Meta ads](https://docs.zernio.com/platforms/meta-ads), [Google Ads](https://docs.zernio.com/platforms/google-ads) |
| Plan execution: edit / pause | Yes: update and pause/resume campaign, ad set, ad; bulk status; budgets and bids; duplicate | Yes: update/pause campaign, keywords, negatives, assets, bidding, ad schedule, targeting; apply Google recommendations | [endpoint catalogue](https://docs.zernio.com/llms.txt) |
| Audiences | Custom, website (1-180 days), lookalike (ratio 0.01-0.20), engagement | `customer_list`, `website` (1-540 days), `lookalike` (narrow/balanced/broad) | [create-ad-audience](https://docs.zernio.com/ad-audiences/create-ad-audience.mdx) |
| Customer-list upload (e.g. Shopify customers) | Yes: email + phone, SHA-256 hashed server-side, 10,000 users per request | Google platform page says Customer Match upload is supported (Data Manager API), but the users-upload endpoint page does not list Google. Verify (UNCONFIRMED) | [add-users](https://docs.zernio.com/ad-audiences/add-users-to-ad-audience.mdx), [Google Ads](https://docs.zernio.com/platforms/google-ads) |
| Conversion events (server-side) | Yes: Meta Conversions API via `POST /v1/ads/conversions`; destination = pixel/dataset ID; dedup via `eventId`; test code; Event Match Quality read; any DENIED consent flag enables Limited Data Use; 1000 events per request | Yes: offline + enhanced conversions via Data Manager `ingestEvents`; destination = conversion action resource name; gclid/gbraid/wbraid or hashed email/phone; EEA/UK consent block; 2000 per request; `eventName` ignored | [send-conversions](https://docs.zernio.com/conversions/send-conversions), [Google Ads](https://docs.zernio.com/platforms/google-ads) |

## Auth and end-user connection flow

- Zernio API: Bearer API key. Keys can be scoped to one profile, read-only, expiring, or with resource groups (including "ads") disabled. [multi-tenant](https://docs.zernio.com/multi-tenant)
- Meta: `GET /v1/connect/{facebook|instagram}/ads` creates a dedicated `metaads` account. Classic mode reuses the parent Facebook posting account's token (needs `ads_management`, `ads_read`) or runs an OAuth round trip. Optional Facebook Login for Business mode uses a System User token with `permissionLevel` full or advertise, completed by a Business admin. Other scopes in the Meta docs: `pages_manage_ads`, `leads_retrieval`, and `business_management` for creating ad accounts. State expires after 30 minutes. [connect-ads](https://docs.zernio.com/connect/connect-ads.mdx), [Meta ads](https://docs.zernio.com/platforms/meta-ads)
- Meta Page and Instagram: a Facebook Page is required (a user with no Page fails with `no_facebook_pages`); ads publish as the Page. Instagram placements need a professional account linked to the Page; accounts connected via Instagram Login cannot run ads. A `metaads` account cannot read organic Page/Instagram insights. [connect-ads](https://docs.zernio.com/connect/connect-ads.mdx)
- Google: own OAuth, separate from YouTube/Business Profile, via `GET /v1/connect/googleads/ads`. Scopes: full Google Ads API plus Data Manager API. Zernio operates under its own approved developer token, so Rita needs no MCC or developer token; MCC-managed accounts are supported. [Google Ads](https://docs.zernio.com/platforms/google-ads)
- Marketing page claims "No Meta App setup required" and "approved partner"; formal Meta Marketing Partner status is not stated. [zernio.com/meta-ads](https://zernio.com/meta-ads)
- Meta ad accounts created via API have no payment method; the customer must add one in Ads Manager. [Meta ads](https://docs.zernio.com/platforms/meta-ads)

## Multi-tenant model

One Zernio profile per customer (a Rita Project), created with `POST /v1/profiles` using the Project id as `externalId` (idempotent; 409 on duplicate). All of that customer's accounts connect under the profile. Webhooks (`account.connected`, `account.disconnected`) carry `externalId`; up to 50 webhook endpoints. Headless connect mode lets Rita build its own account-selection screens. `GET /v1/usage?groupBy=profile` gives per-Project cost. Offboarding requires disconnecting accounts before deleting the profile. White-label branding of the OAuth screen is not described (UNCONFIRMED). [multi-tenant](https://docs.zernio.com/multi-tenant)

## Rate limits

- API, per team by connected-account count: 60 req/min (0-2 accounts), 600 (3-2,000), 1,200 (2,001+). Analytics endpoints: 6 / 10 / 20 req/s. Headers `X-RateLimit-*`; `Retry-After` on 429. One key serves all Projects, so the limit is shared. [rate-limits](https://docs.zernio.com/guides/rate-limits)
- Google: one developer token shared across all Zernio customers (Standard access, no stated daily cap); 429 `quotaExhausted` scoped DEVELOPER or ACCOUNT; Zernio does not retry. User-driven live Google calls (GAQL, search terms, writes): 15 req/min per user. [rate-limits](https://docs.zernio.com/guides/rate-limits), [Google Ads](https://docs.zernio.com/platforms/google-ads)
- Meta: platform 429s surface with a retry delay; no Zernio-specific Meta numbers stated.

## Pricing and per-Project cost

- No plan tiers. Per connected account per month, graduated: accounts 1-2 free, 3-10 $6 each, 11-100 $3, 101+ $1. Ad accounts count toward the total (a `metaads` account counts once regardless of how many ad accounts it reaches). [pricing](https://zernio.com/pricing), [Meta ads](https://docs.zernio.com/platforms/meta-ads)
- Ads: pricing page says managed ads are metered per active ad with the first 500 free; the per-ad rate is NOT stated (UNCONFIRMED). Docs say ads are included on usage-based billing (legacy plans get 403); teams with no payment method are capped at 500 free live ads. Marketing pages say ads are "included on every account" while another page describes an add-on; these conflict and need a direct answer from Zernio.
- Rough estimate (assumption: one `metaads` plus one `googleads` account per Project): about $6-$12 per Project per month beyond the two free accounts, falling to $3 per account past 10 accounts, before per-ad metering. Rita pays this, not the user.

## Gaps vs the direct Meta/Google APIs

- Only Zernio's exposed endpoints are available. Live bulk reads cover Meta campaigns and ad sets only (other platforms 501); Google offers GAQL passthrough, and Meta has async report runs and a flexible live insights query.
- Google: reach 0, empty `actions`, no demographic breakdowns on the analytics endpoint (use GAQL).
- Meta creation success does not mean delivery (review is separate); ad accounts cannot be deleted; ad-account creation is not idempotent.
- Conversion events are batch-oriented; Rita must supply `eventId`, plaintext PII and consent flags. Meta chunks are all-or-nothing, so one malformed event fails its chunk.
- Meta reach is cached up to about an hour.
- Colombia-specific constraints (currency, tax, ad account availability): not addressed in the docs (UNCONFIRMED).

## Risks

1. Cost opacity: per-active-ad price is unpublished; confirm before setting Paywall pricing.
2. Shared limits: one API key and one Google developer token shared across all Projects and other Zernio customers; 600 req/min is ample for MVP but the Google quota is outside Rita's control.
3. Vendor dependency: Rita has no direct Meta/Google app relationship; outages or terms changes hit every Project. Marketing claims (uptime, SOC 2, partner status) are unverified.
4. Meta Page requirement and reconnect flows (`reconnect_required`, `reconnect_mismatch`) will hit wizard step 6; Instagram-Login-only accounts cannot run ads.
5. New Meta ad accounts have no payment method; users must set billing in Ads Manager.
6. 202 backfill responses mean the free Diagnosis may see incomplete history right after connecting; the UI must retry.
7. Google conversion quality depends on click IDs (Shopify must capture gclid) and EEA/UK consent handling.

## Sources

- https://docs.zernio.com and https://docs.zernio.com/llms.txt (endpoint catalogue)
- https://docs.zernio.com/multi-tenant
- https://docs.zernio.com/guides/rate-limits
- https://docs.zernio.com/platforms/google-ads
- https://docs.zernio.com/platforms/meta-ads
- https://docs.zernio.com/connect/connect-ads.mdx
- https://docs.zernio.com/ad-insights/get-ad-analytics.mdx
- https://docs.zernio.com/ad-campaigns/get-ad-tree.mdx
- https://docs.zernio.com/ad-accounts/get-ad-account-live-entities.mdx
- https://docs.zernio.com/ad-audiences/create-ad-audience.mdx
- https://docs.zernio.com/ad-audiences/add-users-to-ad-audience.mdx
- https://docs.zernio.com/conversions/send-conversions
- https://zernio.com/pricing
- https://zernio.com/meta-ads
- https://zernio.com/google-ads
