# Research: Shopify API access for the CRM cache and conversions

Ticket: danyelangel/rita#3. Sources are Shopify and Meta/Google first-party docs, fetched 2026-10-14. Items marked **UNCONFIRMED** were not verifiable from the docs fetched.

## Summary

- Build a **public app** using the **GraphQL Admin API** with **offline (expiring) access tokens**. Scopes: `read_customers`, `read_orders`, `read_all_orders`, `read_products`, `read_inventory`, `read_locations`, and `write_content` later.
- The **60-day order-history default** is the key constraint: full history needs `read_all_orders`, which Shopify must approve separately. Without it the cache can only backfill 60 days.
- Initial backfill: **bulk operations** (JSONL). Incremental sync: **webhooks plus an `updated_at` reconciliation job**, since delivery is not guaranteed.
- Reading customer name, email, phone or address (everything a CRM cache needs) is **protected customer data level 2**, requiring Shopify review for a public app.
- Server-side conversions: send paid orders to Meta CAPI and Google (enhanced conversions or offline click upload), deduplicated by `event_id` (Meta) and `order_id` (Google). Click IDs must be captured at the storefront, because the Order object is not documented to expose them.

## 1. App type, auth, scopes

**App types and tokens**
- Public apps: Shopify approval is required for protected customer data (levels 1 and 2). Custom apps always have level 1 and 2 access. Admin-created custom apps always have level 1, with level 2 depending on the merchant plan. Admin-created custom apps "can no longer be created" per the auth overview. Sources: https://shopify.dev/docs/apps/launch/protected-customer-data and https://shopify.dev/docs/apps/build/authentication-authorization
- Authorization methods: token exchange (embedded in admin), authorization code grant (standalone/outside admin), client credentials grant (only for stores within your own organization), managed installation (scopes set in `shopify.app.toml`). Source: https://shopify.dev/docs/apps/build/authentication-authorization
- Offline tokens persist across sessions and suit webhooks and scheduled jobs; online tokens are tied to a staff session. Rita needs **offline** tokens. Source: same page.
- Public apps must use **expiring offline tokens** for Admin GraphQL: access token 1 hour, refresh token 90 days; each refresh returns a new pair that must be stored. New public apps cannot use non-expiring offline tokens; existing public apps must migrate by **2027-01-01**. The requirement does not apply to custom/merchant-created apps. Source: https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/offline-access-tokens
- Implication: a **public app** is the only realistic path for installing on arbitrary merchant stores. It means refresh-token handling, protected data review and mandatory compliance webhooks. A custom app per merchant avoids review but is not a scalable onboarding flow (and the creation path is deprecated). This is a product decision for the orchestrator.

**Scopes** (source: https://shopify.dev/docs/api/usage/access-scopes)

| Need | Scope |
|---|---|
| Customers | `read_customers` |
| Orders | `read_orders` (last 60 days only), plus `read_all_orders` for full history (requires Shopify approval via Partner Dashboard "API access requests") |
| Products | `read_products` |
| Inventory | `read_inventory`, plus `read_locations` for Location |
| Content write (later) | `write_content` (Article, Blog, Comment, Page); `write_online_store_pages` is an accepted alternative for pages |

## 2. Historical depth and bulk export

- Default order access covers only orders created within the **last 60 days**; older orders need `read_all_orders`. Source: access-scopes page above.
- GraphQL bulk operations (`bulkOperationRunQuery`) export large datasets as **JSONL**; bulk execution does not count against rate limits (only create/poll/cancel calls do). The result URL is signed and **expires after one week**; a query must finish within **10 days**. Limits: up to five bulk queries per shop concurrently on API 2026-01+, one per type on earlier versions. Query restrictions: must contain a connection, max five connections, nested connections at most two levels deep, no top-level `node`/`nodes`. Child records carry `__parentId`. Source: https://shopify.dev/docs/api/usage/bulk-operations/queries
- Normal paginated queries are capped at **25,000 objects**, so large backfills must use bulk operations. Source: https://shopify.dev/docs/api/usage/limits
- Completion: subscribe to the `BULK_OPERATIONS_FINISH` webhook, but delivery is not guaranteed, so also poll `bulkOperation(id:)`. Source: bulk operations page.

## 3. Webhooks for incremental sync

- Delivery to HTTPS, Google Pub/Sub or Amazon EventBridge; subscriptions via `shopify.app.toml` or the GraphQL Admin API; filters and `include_fields` can narrow payloads. Source: https://shopify.dev/docs/apps/build/webhooks
- Delivery "isn't always guaranteed" and **ordering is not guaranteed**. Shopify's guidance: sequence by `X-Shopify-Triggered-At` or payload `updated_at`, dedupe with `X-Shopify-Webhook-Id`, verify `X-Shopify-Hmac-Sha256`, and run **reconciliation jobs** that fetch records updated since the last run. Source: same page.
- Topics the cache needs: `orders/create`, `orders/paid`, `orders/updated`, `customers/create`, `customers/update`, `products/update`, `inventory_levels/update`, `app/uninstalled`. Topic names are on the webhooks reference (https://shopify.dev/docs/api/webhooks), but the fetch was truncated, so **exact per-topic scopes and the retry policy are UNCONFIRMED**.
- Public apps in the App Store must subscribe to the mandatory compliance topics (`customers/data_request`, `customers/redact`, `shop/redact`). Source: https://shopify.dev/docs/api/webhooks. The `shop/redact` entry itself was not visible in the fetch (**UNCONFIRMED** detail).

## 4. Rate limits

- GraphQL Admin uses calculated query cost with a leaky bucket. Restore rate per plan: Standard 100 points/s, Advanced 200, Plus 1000, enterprise 2000. **Max single query cost 1,000 points.** Bucket size per plan is **UNCONFIRMED** (the page shows only a sample `maximumAvailable: 1000`). Source: https://shopify.dev/docs/apps/build/apis/graphql-admin/rate-limits
- Back off at least one second on throttling; Shopify may lower limits temporarily. Array inputs max 250 items. Source: https://shopify.dev/docs/api/usage/limits
- Design consequence: bulk operations for backfill (not rate-limited), webhooks for deltas, small reconciliation queries. This is exactly the "avoid excess API calls" goal of the cache.

## 5. Protected customer data

Source: https://shopify.dev/docs/apps/launch/protected-customer-data
- No access by default. Level 1 = customer data excluding name, address, phone, email. Level 2 = includes name, address, phone or email. Each of name, address, email and phone is requested separately.
- Public apps need Shopify review for both levels; development-only apps can skip review. Process: Partner Dashboard > app > API access requests > Protected customer data access, state reasons, complete data protection details, submit app for review. Shopify may run deeper data protection reviews (more likely with many installs, large volumes, more fields, long retention).
- Level 1 requirements: minimum data, merchant disclosure, purpose limitation, consent and opt-out handling, privacy agreements, retention periods, encryption at rest and in transit. Level 2 adds: encrypted backups, test/production data separation, data loss prevention strategy, limited staff access, strong staff passwords, access log of protected data, incident response policy.
- Impact on Rita: a CRM cache stores exactly the level 2 fields, and conversion matching (hashed email/phone) needs them. Plan for the review and for the level 2 controls (access log, retention policy, encryption) in the build.
- The API will not return protected data on non-development stores until the app is configured and approved (access-scopes page).

## 6. Paid orders as server-side conversions

**Which orders:** use `orders/paid` (or `displayFinancialStatus` plus `processedAt`), skip `test: true` orders. Order fields available: `totalPriceSet`, `currencyCode`, `processedAt`, `discountCodes`, `sourceName`, `clientIp`, `customerJourneySummary`. Source: https://shopify.dev/docs/api/admin-graphql/latest/objects/Order

**Attribution data:** `customerJourneySummary` gives first/last visit and moments (needs `read_orders`; check `ready` first, because it is null while attribution is in progress; 30-day attribution window). Its docs do not define UTM or click-ID fields. `landingSite`/`referringSite` were not found as named fields (only deprecated `landingPageUrl`, `referrerUrl`). **Whether `fbclid`/`gclid` are retrievable from the Order object is UNCONFIRMED**; plan on capturing them at the storefront (web pixel / customer events) or from visit landing URLs. Source: https://shopify.dev/docs/api/admin-graphql/latest/objects/CustomerJourneySummary

**Meta Conversions API**
- Customer params: `em`/`ph` SHA-256 hashed (normalize: trim and lowercase email; strip symbols and leading zeros from phone, include country code); `fbc`, `fbp`, `client_ip_address`, `client_user_agent` are not hashed; `client_ip_address` and `client_user_agent` must be supplied manually on server events; `external_id` recommended. `fbc` format `fb.{subdomain_index}.{creation_time}.{fbclid}`. Source: https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/customer-information-parameters
- **Dedup:** only needed if the same event is also sent by the Pixel. Primary method: same `event_name` and `event_id` (Pixel `eventID`) to the same Pixel ID within **48 hours**; the first received is kept. Fallback: `fbp`/`external_id` plus event name, which only dedupes when the browser event arrives first. If Rita sends server-only, dedupe against the store's own Pixel/channel app by using a deterministic `event_id` (e.g. the Shopify order ID) and avoiding double-sending. Source: https://developers.facebook.com/docs/marketing-api/conversions-api/deduplicate-pixel-and-server-events

**Google Ads**
- Enhanced conversions for web: send hashed first-party data (SHA-256; email trim and lowercase, with dot/plus removal for gmail.com and googlemail.com) as an adjustment keyed by **`order_id`** that must match the transaction ID the Google tag sent; include `gclid` when available; adjustments must be sent within **24 hours** of the conversion; max five `UserIdentifier` objects per adjustment; customer data terms must be accepted (`accepted_customer_data_terms`). Source: https://developers.google.com/google-ads/api/docs/conversions/upload-online
- The fetched page text was truncated at 100,000 characters (final ~16k not read), so details of offline click-conversion upload (the non-tag path) are **UNCONFIRMED** here; that is the path if the store has no Google tag order IDs.
- Google's page does not use the term "dedup"; matching by order ID is the mechanism.

**Shopify-side note:** Shopify web pixels/customer events run in a sandbox and honor customer consent (https://shopify.dev/docs/apps/build/marketing-analytics/pixels). The fetched content did not cover `checkout_completed` or the Meta/Google channel apps' server-side behavior, so whether a merchant's existing channel app already sends CAPI events (a duplicate risk) is **UNCONFIRMED**. Open question for the Meta/Google connection tickets: detect and avoid double-sending.

Zernio (the planned ad-connection layer) was not researched here; whether it exposes CAPI/offline-conversion endpoints is out of this ticket.

## 7. Colombia notes

- Indirect evidence only, **UNCONFIRMED**: Shopify Payments appears not to support Colombian merchants (not in the supported-countries list found), so Colombian stores likely use third-party gateways, and the payment status on orders then depends on the gateway. Check https://help.shopify.com/en/manual/payments/shopify-payments/supported-countries.
- COP appears among currencies Shopify treats as volatile: it is not auto-activated and requires a market per country (https://help.shopify.com/en/manual/international/pricing/volatile-currencies). Rita must read `currencyCode` and `presentmentCurrencyCode` per order rather than assume one currency, and convert conversion values as the ad platforms require.
- Shopify checkout asks Colombian customers for extra tax fields (from search results; detail UNCONFIRMED). Phone numbers need the +57 country code before hashing for Meta.
- Shopify's protected data requirements apply regardless of country. Colombian data-protection law (Ley 1581 de 2012) is not a Shopify doc and was not researched here; flag for the legal/privacy discussion.

## Decisions this enables (for the orchestrator)

1. Cache scope: customers, orders (full history if `read_all_orders` is granted, else 60 days), products, inventory levels/locations.
2. Sync: bulk-operation backfill, webhook deltas, scheduled `updated_at` reconciliation, with idempotency by webhook ID and `updated_at` ordering.
3. Start the `read_all_orders` and protected customer data (level 2) requests early; they gate full history and CRM fields, and public-app review timing is not documented (**UNCONFIRMED**).
4. Conversions: `orders/paid` as trigger, deterministic IDs for dedupe (`event_id` for Meta, `order_id` for Google), storefront capture of `fbc`/`fbp`/`gclid`.
