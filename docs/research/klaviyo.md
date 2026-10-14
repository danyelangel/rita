# Research: Klaviyo API coverage (issue #4)

Researched 2026-10-14 from Klaviyo's official developer docs and help center. Items I could not confirm from a primary source are marked **UNCONFIRMED**. Note: `GLOSSARY.md` does not exist in this repo yet; terms used here (Diagnosis, Plan, Approval, Goal, Email Marketing) follow the map issue #1.

## Answer in one paragraph

Rita can do nearly everything the MVP needs through the public API: read campaigns, flows, segments, metrics and performance reports; create segments, templates and draft campaigns; schedule and cancel campaign sends; and switch flow status. The main gaps are (1) **no API to edit an existing flow's definition** (only create new flows, and change status), (2) **no API to configure the Klaviyo-to-Meta audience sync** (UI-only), (3) the **Reporting API is tightly rate-limited (2/min, 225/day)** and capped at 1 year per request, so the Diagnosis must budget its calls, and (4) **24 months of history is not confirmed**. Use **OAuth** (not private keys) for a multi-tenant product.

## 1. Auth and scopes

- **Private API key** (`Authorization: Klaviyo-API-Key pk_...`): per-account key with Custom / Read-Only / Full Access scope; scopes cannot be edited after creation; only Owner/Admin/Manager can create. Private-key integrations share one rate-limit quota per account. <https://developers.klaviyo.com/en/docs/authenticate_>
- **OAuth 2.0 authorization code with PKCE (S256)**, required for all clients. Apps are created in Klaviyo by Owner/Admin/Manager roles; redirect URIs are allowlisted; scopes are fixed per app and Klaviyo rejects apps requesting more than needed. Access token lifetime 1 hour (subject to change; use `expires_in`); refresh tokens last until uninstall but are revoked after 90 days unused; refresh limited to 10/min; token endpoint is `https://a.klaviyo.com/oauth/token`. The code expires in 5 minutes. A listed App Marketplace entry needs OAuth, 5 active installs and review, but an unlisted app can be installed via a shared install URL. <https://developers.klaviyo.com/en/docs/set_up_oauth>
- **Rate-limit benefit of OAuth:** each installed app instance gets its own quota per account. <https://developers.klaviyo.com/en/docs/rate_limits_and_error_handling>
- **Scopes Rita needs:** `campaigns:read`, `campaigns:write`, `flows:read`, `flows:write`, `segments:read`, `segments:write`, `templates:read`, `templates:write`, `metrics:read`, `events:read`, `lists:read`, `profiles:read`, `accounts:read`. Reporting uses `campaigns:read`, `flows:read`, `forms:read`, `segments:read`. Full scope list: <https://developers.klaviyo.com/en/docs/authenticate_>
- **Versioning:** every call sends a `revision` date header; a revision is supported about 2 years (1 stable + 1 deprecated). Retired revisions fall forward unless opted out. <https://developers.klaviyo.com/en/docs/api_versioning_and_deprecation_policy>
- **UNCONFIRMED:** whether an unlisted OAuth app has any approval step or install cap beyond what the OAuth page says (the page describes none).

## 2. Rate limits

Tiers (burst per second / steady per minute): XS 1/15, S 3/60, M 10/150, L 75/700, XL 350/3500; 429 responses carry `Retry-After`. <https://developers.klaviyo.com/en/docs/rate_limits_and_error_handling>

| Endpoint | Limit | Source |
|---|---|---|
| Campaign values report | 1/s, 2/min, **225/day** | <https://developers.klaviyo.com/en/reference/query_campaign_values> |
| Flow values report | 1/s, 2/min, **225/day** | <https://developers.klaviyo.com/en/reference/query_flow_values> |
| Get Campaigns / Create Campaign | 10/s, 150/min | <https://developers.klaviyo.com/en/reference/get_campaigns>, <https://developers.klaviyo.com/en/reference/create_campaign> |
| Create Campaign Send Job | 10/s, 150/min | <https://developers.klaviyo.com/en/reference/create_campaign_send_job> |
| Update Flow (status) | 3/s, 60/min | <https://developers.klaviyo.com/en/reference/update_flow> |
| Get Segments | 75/s, 750/min | <https://developers.klaviyo.com/en/reference/get_segments> |
| Create Segment | 1/s, 15/min, **100/day** | <https://developers.klaviyo.com/en/reference/create_segment> |
| Create Template | 75/s, 750/min | <https://developers.klaviyo.com/en/reference/create_template> |
| Query Metric Aggregates | 3/s, 60/min | <https://developers.klaviyo.com/en/reference/query_metric_aggregates> |
| Get Events | 350/s, 3500/min | <https://developers.klaviyo.com/en/reference/get_events> |

## 3. Reads (Diagnosis)

- **Campaigns:** Get Campaigns requires a `messages.channel` filter (email, sms, mobile_push); filters on status, name, created/scheduled/updated dates; messages and templates are retrievable. <https://developers.klaviyo.com/en/reference/campaigns_api_overview>
- **Flows:** list, get one (add `additional-fields[flow]=definition` for the definition), actions, messages, message templates. <https://developers.klaviyo.com/en/reference/flows_api_overview>
- **Segments:** list/get with definition; filter by `is_active`, name. <https://developers.klaviyo.com/en/reference/segments_api_overview>
- **Metrics and events:** Get Metrics, Query Metric Aggregates (hour/day/week/month intervals), Get Events (filter by metric, profile, datetime; cursor paged). <https://developers.klaviyo.com/en/reference/metrics_api_overview>
- **Performance (opens, clicks, conversions, revenue, revenue per recipient):** only through the Reporting API (`campaign-values-reports`, `flow-values-reports`, `flow-series-reports`, `segment-*`, `form-*`); the campaigns API itself returns no performance data. `conversion_metric_id` is required (e.g. the Placed Order metric ID). <https://developers.klaviyo.com/en/reference/reporting_api_overview>
- **Historical depth:** each Reporting request timeframe is capped at **1 year**. Whether data older than 24 months is reachable, and whether there is any retention cap on events or aggregates, is **not stated** in the docs I read. **UNCONFIRMED.** Plan: a 24-month look-back is two reporting windows per report type; budget against the 225/day limit and verify against a real account.
- Aggregates group by event time, not send date, so the Reporting API is the right source for campaign/flow performance.

## 4. Writes (plan execution)

- **Segments:** create (POST) and update (PATCH). Definitions are AND of groups, OR within a group; seven condition types (list membership [one list only], consent, metric count/sum, postal distance, predictive analytics, property, region); max 5 segments processing at a time and 100 created per day. Delete is only mentioned, no endpoint documented. <https://developers.klaviyo.com/en/reference/segments_api_overview>
- **Templates:** create, update, clone, delete, render. Editor types `CODE` and `USER_DRAGGABLE` take HTML; `SYSTEM_DRAGGABLE` takes a `definition`. Universal content blocks cannot be referenced (inline the HTML). Limit 1,000 templates per account. <https://developers.klaviyo.com/en/reference/templates_api_overview>
- **Campaigns:** create (message and audience in one payload), update while draft, clone, assign template, create recipient estimation job, create send job (schedules or sends), update send job (revert to draft or cancel). Send strategies: immediate, static (optionally recipient-local), throttled, smart send time (revision 2026-04-15 or later, audience of at least 12,000). Email campaigns need an assigned template before scheduling. Cancelled campaigns must be cloned to resend. Campaigns do not send to unconsented profiles. <https://developers.klaviyo.com/en/reference/campaigns_api_overview>, <https://developers.klaviyo.com/en/reference/create_campaign>
- **Flows:** can **create** a new flow from an encoded definition (always created as Draft) and **change status** (`draft`, `manual`, `live`, which also updates its actions). **No documented endpoint to edit an existing flow's definition or messages.** Template content used inside flow messages can be updated through the Templates API. Klaviyo advises against pre-creating flows in customer accounts. <https://developers.klaviyo.com/en/reference/flows_api_overview>, <https://developers.klaviyo.com/en/reference/update_flow>

## 5. Shopify integration events (data Rita can read from Klaviyo)

Synced by the native integration: Checkout Started, **Placed Order**, Ordered Product (one per item), Fulfilled Order, Fulfilled Partial Order, Cancelled Order, Refunded Order (full refunds only); shipping metrics for some carriers; onsite metrics via Klaviyo snippet (Active on Site, Viewed Product, Viewed Collection, Submitted Search, Added to Cart). On connect it syncs the last 90 days first, then full history (minutes to days), then real time. Caveats: events are not re-synced after creation; Klaviyo revenue does not subtract cancels/refunds, so it can differ from Shopify (Shopify stays the source of truth for conversions per map #1); Ordered Product quantity is always 1; Placed Order and Ordered Product arrive seconds apart (use Placed Order for triggers and filters). <https://help.klaviyo.com/hc/en-us/articles/115005080447>

## 6. What the public API does not expose

- **Klaviyo-to-Meta audience sync:** configured only in the Klaviyo UI (one-to-one list or segment to Meta custom audience, hourly membership updates, 24-48h Meta lag, minimum 100 profiles, only emails tied to a Facebook login; lead ads sync only lists). I found no API endpoint to create or manage it; Klaviyo's developer guide for ad platforms describes the reverse pattern (pull `GET /api/segments/{id}/profiles`, diff, push to the ad platform yourself). **UNCONFIRMED** that no undocumented or partner endpoint exists. Implication: Rita can create the segment but cannot create the sync; a Meta audience would go through Zernio or be a manual user step. <https://help.klaviyo.com/hc/en-us/articles/360040954011>, <https://developers.klaviyo.com/en/docs/guide_to_integrating_an_ad_platform>
- Editing an existing flow's definition (see section 4).
- Deleting segments (no endpoint documented), deleting campaigns (mentioned but undocumented).
- Performance metrics outside the Reporting API.

## 7. Deliverability prerequisites

- Set up a **branded sending domain** (DKIM and SPF handled by Klaviyo's CNAME/NS records); the domain must be registered 30+ days and have prior sending history. From-address root domain must align with the sending domain.
- **DMARC** is a DNS TXT record the merchant owns; `p=none` meets initial Gmail/Yahoo requirements; Klaviyo can generate the record but cannot publish it.
- Gmail/Yahoo bulk-sender rules apply at 5,000+ daily emails to Gmail; Klaviyo recommends the branded domain and DMARC for accounts above 5,000 profiles. One-click unsubscribe headers are added by Klaviyo; a visible unsubscribe link is still needed in the body.
- Rita should treat these as Diagnosis checks (read-only guidance); it cannot set DNS. Sources: <https://help.klaviyo.com/hc/en-us/articles/4402601857307>, <https://help.klaviyo.com/hc/en-us/articles/115000357752>, <https://help.klaviyo.com/hc/en-us/articles/25620771311643>
- **UNCONFIRMED:** an API endpoint that returns sending-domain or DMARC status (not found; search covered help articles only).

## 8. Coverage matrix

| Rita need | Covered? | How / constraint |
|---|---|---|
| Diagnosis: list campaigns and content | Yes | Get Campaigns (channel filter required), messages, templates |
| Diagnosis: campaign performance | Yes, limited | Reporting API only; 2/min, 225/day, 1-year window |
| Diagnosis: flows and their definitions | Yes | Get Flows with `definition`; actions and messages |
| Diagnosis: flow performance | Yes, limited | Flow values/series reports; same limits |
| Diagnosis: segments and sizes | Yes | Get Segments; segment reports for membership trends |
| Diagnosis: revenue and orders from Shopify events | Yes, with caveats | Metric aggregates / events (Placed Order etc.); prefer Shopify for truth |
| Diagnosis: 24-month history | **Unconfirmed** | 1-year max per request; no stated retention |
| Diagnosis: deliverability posture | Partial | Domain/DMARC guidance only; no confirmed status endpoint |
| Execute: create segment | Yes | 100/day cap, limited condition types |
| Execute: create/update template | Yes | HTML editor types; no universal-content block refs |
| Execute: draft campaign | Yes | Create Campaign + assign template |
| Execute: schedule / cancel campaign | Yes | Send Job create/update; clone to resend |
| Execute: new flow | Yes | Created as Draft |
| Execute: edit existing flow | **No** | No endpoint documented; only status change and template updates |
| Execute: turn flow live/draft | Yes | PATCH flow status |
| Execute: Klaviyo-to-Meta audience sync | **No** | UI-only; route via Zernio or manual |
| Auth for many merchants | Yes | OAuth + PKCE, per-install quota |

## 9. Implications for the map

- The Plan can propose "edit this flow" only as "create a replacement draft flow" or "update its template content", or as a manual step for the user.
- Diagnosis must be designed around the Reporting quota (cache results; batch by campaign set; do not re-run per page load).
- Audience sync to Meta belongs on the Performance Marketing side (Zernio), not Klaviyo.
