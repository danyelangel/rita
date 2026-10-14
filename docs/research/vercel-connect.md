# Research: Vercel Connect for Shopify and Klaviyo authentication

Ticket: danyelangel/rita#16. Researched 2026-10-14 from Vercel's official docs (pages were fetched through a summarising tool; quoted facts are from page text, gaps are flagged as UNCONFIRMED).

## Recommendation

**Do not build the MVP's Shopify and Klaviyo authentication on Vercel Connect. Handle OAuth ourselves, behind a thin token-provider interface.** Revisit Connect after a spike if it is wanted for convenience.

Reasons, in order of weight:

1. **It does not remove the hard parts.** The Shopify connector is a *preset*, and "preset connectors don't register your app with the external service for you" ([changelog: preset connectors](https://vercel.com/changelog/vercel-connect-preset-connectors)). We still register our own Shopify public app and own the protected-customer-data and `read_all_orders` approvals (see #3). The Klaviyo entry in the catalog is tagged **API Key**, not OAuth ([connector catalog](https://vercel.com/connect/browse)), which would not give a Klaviyo OAuth flow for end users.
2. **Multi-tenancy fit is unproven.** Rita has one Shopify store and one Klaviyo account per Project, with many Projects. Connect's tenancy primitive ("installations") is documented for Slack, GitHub, Linear and Microsoft; "Snowflake, Salesforce, API Key, and Custom OAuth connectors reach exactly one account, so there's nothing to install" ([installations](https://vercel.com/docs/connect/concepts/installations)). Whether a Shopify preset can hold one grant per shop is not documented. UNCONFIRMED.
3. **Cost and coupling for little gain.** Every call to `getToken` is a billed token request (Pro: $3.00 per 1,000), with a 200 requests/minute/team rate limit on `getToken` ([pricing](https://vercel.com/docs/connect/pricing), [limits](https://vercel.com/docs/connect/limits)). A CRM cache sync across many shops would sit on that. Standard OAuth refresh is something we code once and then own.

## How the connect flow works

- Setup is team-level: create a connector once (dashboard or `vercel connect create shopify --name acme-shop`), link it to projects and environments, then call `getToken('shopify/acme-shop', { subject: ... })` ([Connect overview](https://vercel.com/docs/connect), [Shopify connector page](https://vercel.com/connect/shopify)). Preset creation currently opens the dashboard to finish setup ([changelog](https://vercel.com/changelog/vercel-connect-preset-connectors)).
- Caller authentication: the deployment's Vercel OIDC token (project and environment checked against project links), or a Vercel access token off-platform ([authentication](https://vercel.com/docs/connect/concepts/authentication)).
- End user consent: a `{ type: 'user', id }` subject that has not authorized throws `UserAuthorizationRequiredError`; the app must surface a "connect your account" UI and trigger the consent flow ([tokens](https://vercel.com/docs/connect/concepts/tokens)). The docs do not specify a consent URL or redirect helper, whether `id` is our own user id, or whether end users need Vercel accounts. UNCONFIRMED (the SDK reference only shows `id` and an optional `issuer`, [ts-sdk-reference](https://vercel.com/docs/connect/ts-sdk-reference)).

## Who registers the OAuth client

Per [Connect overview](https://vercel.com/docs/connect#connector-ownership-models) and [product terms](https://vercel.com/docs/connect/legal):

- *Vercel Managed Connector*: Vercel registers the client. Listed for Slack, GitHub, Linear, Microsoft, Snowflake, Salesforce only. Shopify and Klaviyo are not on that list.
- *Customer Managed Connector*: we register the client and give Vercel the credentials; Vercel stores them and exchanges tokens.
- Preset (Shopify): does not register our app; we must add the credentials the service requires ([changelog](https://vercel.com/changelog/vercel-connect-preset-connectors)). So for Shopify: **we register our own public app** (already required by #3).
- Klaviyo: catalog tag is API Key only. Whether there is an OAuth method is UNCONFIRMED. The ticket describes Klaviyo as "experimental, unverified"; the catalog uses that notice for many entries ("This connector provider has not been verified by Vercel") but I could not confirm it for Klaviyo specifically, and the Klaviyo connector page itself does not mention it.

## Tokens: issue, refresh, revoke

- Short-lived provider tokens issued per request; in-process LRU cache (100 entries), reuse until within a 30 s validity buffer ([tokens](https://vercel.com/docs/connect/concepts/tokens)).
- For OAuth connectors Vercel drives the authorization-code flow and refresh and stores the refresh token; code never sees it ([authentication](https://vercel.com/docs/connect/concepts/authentication)). How the Shopify preset handles Shopify's offline/expiring token model is UNCONFIRMED.
- Revoke: dashboard, CLI, `revokeToken`, or REST. Works only if the provider exposes a revocation endpoint; otherwise Vercel only marks the token for deletion and the provider credential may keep working until it expires ([tokens](https://vercel.com/docs/connect/concepts/tokens), [legal §8](https://vercel.com/docs/connect/legal)). Provider-side uninstall removes the installation ([installations](https://vercel.com/docs/connect/concepts/installations)), but we would still need Shopify's app-uninstall webhook for our own cleanup.

## Scopes

Per request: `scopes` (provider scope strings, or `['*']` for connector defaults), `resources`, `authorizationDetails` ([ts-sdk-reference](https://vercel.com/docs/connect/ts-sdk-reference)). The Shopify example uses `write_products`. New scopes reach a user only after they re-authorize. The scope list actually offered by the Shopify and Klaviyo connectors is not published on their pages. UNCONFIRMED. Shopify's `read_all_orders` and protected customer data are gated by Shopify's approval of *our* app, not by Vercel, so Connect cannot grant them.

## Background jobs without a signed-in user

- The `app` subject acts as the service and needs no user. A `user` subject needs no live session either: the stored grant is refreshed server-side, and requesting a token for an arbitrary user id requires the OIDC token, i.e. a deployed server context ([authentication](https://vercel.com/docs/connect/concepts/authentication)). This suggests cron and workflow jobs on Vercel can fetch per-Project tokens. Not demonstrated in the docs for a Shopify preset. UNCONFIRMED.
- Jobs hit `NoValidTokenError` if the grant was revoked; we must handle that state.
- Off-Vercel workers must use a Vercel access token, which can only request its own user or the `app` subject, so per-Project user tokens would not work there.

## Status, beta, pricing, limits

- **Not beta anymore.** Generally available on all plans since 2026-08-25 ([GA changelog](https://vercel.com/changelog/vercel-connect-ga)). The pricing page still mentions "Connect Beta Customers" with new pricing effective 2026-09-25 ([pricing](https://vercel.com/docs/connect/pricing)). Individual connectors can still carry Beta/Experimental labels in the catalog.
- Pricing: Hobby 500 token requests/month free; Pro $3.00 per 1,000; Enterprise custom. Triggers are separate (Pro $0.95 per 1,000), only for connectors supporting trigger forwarding. No pricing is stated on the Shopify or Klaviyo pages.
- Limits: `getToken` 200 requests/min/team; write operations (incl. `revokeToken`) 50/min/team; a 429 throws in the SDK ([limits](https://vercel.com/docs/connect/limits)).
- Terms: Vercel may add or remove third-party platforms without notice and does not warrant availability; we remain responsible under Shopify's and Klaviyo's developer terms ([legal §2, §7, §9](https://vercel.com/docs/connect/legal)). Regulated data (card data, PHI, etc.) must not be routed through it (§5). Shopify customer data itself does not pass through Connect (only tokens); confirm against Shopify's protected-data requirements.

## Lock-in versus doing OAuth ourselves

- Lock-in: refresh tokens live only on Vercel's infrastructure and we never receive them. No export is documented, so leaving means every merchant re-authorizes. Runtime coupling: token calls need Vercel OIDC (or an access token), add a network hop and a rate limit on every sync. Cost scales with token requests.
- Doing it ourselves: Shopify OAuth (offline token, HMAC validation, shop domain handling) and Klaviyo OAuth (refresh handling) are each small; we need our own Shopify app and webhooks registered regardless. We keep tokens and revocation on uninstall, with no per-call fee or rate limit.

## Risks if we use it anyway

1. Shopify preset may not map one grant per shop for many Projects (UNCONFIRMED).
2. Klaviyo may be API Key only, forcing merchants to paste private keys; no OAuth consent screen (UNCONFIRMED).
3. Shopify webhooks (orders, customers, uninstall, mandatory privacy webhooks) are not handled by Connect for this preset; we need our own endpoint and HMAC verification anyway.
4. Hot-path cost and the 200/min rate limit during backfill and the Monday plan.
5. Provider-side revocation may not work for every connector.
6. Vercel can drop a platform without notice.

## Suggested next step

If Connect is still wanted, run a half-day spike: create a Shopify preset with our dev app, authorize two dev shops under two user ids, fetch tokens from a cron route and call the Admin GraphQL API with them; check Klaviyo connection methods in the dashboard setup. Otherwise implement OAuth directly behind an interface (`getAccessToken(projectId, provider)`) so Connect could be swapped in later.

## Sources

- https://vercel.com/docs/connect
- https://vercel.com/docs/connect/concepts/connectors
- https://vercel.com/docs/connect/concepts/tokens
- https://vercel.com/docs/connect/concepts/installations
- https://vercel.com/docs/connect/concepts/authentication
- https://vercel.com/docs/connect/ts-sdk-reference
- https://vercel.com/docs/connect/pricing
- https://vercel.com/docs/connect/limits
- https://vercel.com/docs/connect/legal
- https://vercel.com/changelog/vercel-connect-ga
- https://vercel.com/changelog/vercel-connect-preset-connectors
- https://vercel.com/connect/browse
- https://vercel.com/connect/shopify
- https://vercel.com/connect/klaviyo
