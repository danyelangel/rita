# Research: dLocal Go for subscriptions in Colombia

Ticket: #17. Researched 2026-10-14 from dLocal Go's official docs (API docs and Help Center). Items marked **UNCONFIRMED** are not stated in any official page we could read; confirm with dLocal Go support or sales before building on them.

Sources:
- API docs index: https://docs.dlocalgo.com/integration-api/llms.txt
- Help Center: https://helpcenter.dlocalgo.com/en/
- dLocal (parent company, not "Go") Colombia page, used only for per-method recurring support: https://docs.dlocal.com/docs/colombia

## Verdict (what the Paywall can and cannot do)

**Can**
- Sell a subscription to Colombian buyers, charged in COP, with the plan priced in COP or in USD (USD is converted to COP at payment).
- Use a hosted checkout (no customization) via a plan `subscribe_url`, or a card-on-file flow via the Payments API with a stored token.
- Cancel a single subscriber, cancel a whole plan, and move a subscriber to another plan (immediate charge) via API.
- Receive a webhook (`notification_url`) per plan, signed with HMAC-SHA256, retried every 10 minutes for up to 30 days.

**Cannot, or not documented**
- **Free trial: no documented way to set one.** The plan response shows `free_trial_days` (example value 0) but Create a Plan has no such request field and no page defines it.
- **Retries of failed subscription charges: not documented.** Only the card-on-file flow says a failed charge returns a `redirect_url` for the customer to retry; the retry policy is left to the merchant.
- **No subscription lifecycle events documented.** Notifications are described only for payment and refund status changes, with a body of just `payment_id`.
- **Card-on-file is explicitly card-only.**
- **No change of payment method on an existing subscription** (cancel and recreate instead).
- **No checkout customization** (hosted checkout).
- Frequencies: DAILY, WEEKLY, MONTHLY, YEARLY (API). The Help Center UI lists weekly, monthly, quarterly, semi-annual, annual. Quarterly/semi-annual presumably map to `frequency_value` 3/6 on MONTHLY (**UNCONFIRMED**).

## 1. Subscriptions and card-on-file in Colombia

Two mechanisms exist:

1. **Subscription plans** (managed plan + `subscribe_url`).
   - Create via `POST /v1/subscription/plan` or the dashboard. Fields: name, description, country (optional; if omitted, checkout asks and currency must be USD), currency, amount, `frequency_type` (DAILY/WEEKLY/MONTHLY/YEARLY), `frequency_value` (default 1), `day_of_month` (MONTHLY only), `max_periods`, `notification_url`, `success_url`, `back_url`, `error_url`. Source: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/subscriptions/create-a-plan.md
   - The `subscribe_url` takes an `email` query param to prefill and an `external_id` (max 255 chars) returned to `success_url`. Source: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/subscriptions.md
   - Plans never expire; cancelling a plan stops charges to all its active subscribers. Sources: https://helpcenter.dlocalgo.com/en/articles/7925896-do-subscriptions-expire , https://helpcenter.dlocalgo.com/en/articles/7925904-how-do-i-cancel-a-subscription
2. **Card-on-file recurring payments** (you own the schedule).
   - Create a payment with `allow_recurring: true`; customer pays on hosted checkout; store the returned `merchant_checkout_token`; later charge with `POST /v1/payments/recurring/{merchant_checkout_token}` (amount, description, orderId). While enabled, only credit and debit cards are offered. Failed charges (insufficient funds, expired card) return a `redirect_url` for the customer to update their method. Source: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/payments/recurring-payments.md
   - The page's only example is Argentina; Colombia is not named there (**UNCONFIRMED for dLocal Go specifically**). Note the Create a Payment reference page does not itself list `allow_recurring`.
   - This is the route for variable amounts, usage-based pricing, or custom trial and dunning logic, at the cost of building them ourselves.

**Colombia specifics.** dLocal Go supports selling in Colombia and a Colombian company can open an account (https://helpcenter.dlocalgo.com/en/articles/7229140-in-which-countries-can-i-sell-with-dlocal-go , https://helpcenter.dlocalgo.com/en/articles/7240022-where-does-my-company-have-to-be-registered-if-i-want-to-open-a-dlocal-go-account). Per dLocal's Colombia page (parent product, same rails, not a dLocal Go page): currency COP, 0 decimals; payer document CC or NIT, 6 to 11 digits; recurring/tokenization supported for Visa, Mastercard, Amex, Diners, Codensa cards; Nequi Push (recurring only via enrollment) and Mercado Pago Wallet Connect (via enrollment) also recurring; PSE, Efecty, Bancolombia, Nequi QR/Mixed are one-shot or not listed as recurring. Source: https://docs.dlocal.com/docs/colombia . Whether dLocal Go's subscription plans use non-card methods like Nequi is **UNCONFIRMED**; the Go card-on-file page says cards only.

## 2. Pricing in COP and USD

- Buyers always pay in their local currency; you receive in your company's country currency. A USD-priced plan is converted to COP at payment; the buyer bears the FX cost. A COP-priced plan makes the merchant absorb FX when settling to USD. Sources: https://helpcenter.dlocalgo.com/en/articles/7169372-what-is-the-exchange-rate-and-how-dlocal-go-apply-it , https://helpcenter.dlocalgo.com/en/articles/7925893-what-happens-if-i-create-a-subscription-in-dollars-and-a-customer-from-my-country-pays-me
- Implication (inference, not stated in docs): a USD plan means the COP charge can drift between renewals as the rate moves. A plan is single-currency and single-country, so one plan per price point per country/currency.
- Minimum sale: USD 1 or local equivalent. Source: https://helpcenter.dlocalgo.com/en/articles/7240013-what-is-the-minimum-transaction-amount-i-can-sell

## 3. Free trial and cancel-before-charge

- Free trial: not configurable per the docs (see verdict). A first charge happens at subscribe time ("Charged on subscription and on each renewal"). **UNCONFIRMED** whether `free_trial_days` can be set by dLocal support or the dashboard.
- Rita's flow has the Diagnosis free and the Paywall after it, so a trial may not be needed. If one is, the workaround is card-on-file (authorize a first payment, charge later) or a low/zero first plan, both needing our own logic.
- Cancel: `PATCH /v1/subscription/plan/:planId/subscription/:subscriptionId/deactivate` sets `active:false` (https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/subscriptions/cancel-subscription.md). Reversibility and proration are not documented. No documented buyer self-cancel portal (**UNCONFIRMED**); the merchant cancels.
- Plan change: `POST .../change-plan?new_plan_id=` moves the subscriber and charges the new plan immediately, no proration documented (https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/subscriptions/modify-a-subscribers-plan.md).

## 4. Hosted versus embedded checkout

- Hosted: plan `subscribe_url` or payment `redirect_url`. dLocal Go Checkout cannot be customized (https://helpcenter.dlocalgo.com/en/articles/7339129-can-i-customize-the-dlocal-go-checkout), so no Rita branding inside.
- Embedded: SmartFields JS SDK for card inputs in our own page (needs `allow_transparent: true` and a separate SmartFields API key requested from support). The docs do not mention recurring or subscriptions for it and show only Argentina examples. Source: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/transparent-checkout.md . Treat embedded plus recurring in Colombia as **UNCONFIRMED**.

## 5. Webhooks and lifecycle events

- One `notification_url` per payment or plan. POST body is only `{"payment_id": ...}`; we must then call Retrieve payment for status. Signed `V2-HMAC-SHA256` in the Authorization header over `ApiKey + payload` using the secret key. Non-200 triggers retry every 10 minutes for up to 30 days. Source: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/payments/notifications.md
- Documented only for payment and refund changes. No subscription-created/cancelled/past-due events are documented, so derive state by polling subscriptions and executions.
- Execution statuses: PENDING, COMPLETED, DECLINED; subscription status example `CONFIRMED` (full enum not documented). Source: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/subscriptions/retrieve-all-the-executions-for-a-subscription.md
- Rate limit: 100 requests/minute per IP per endpoint, 429 plus one-minute cooldown (https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/rate-limits.md). Polling must be modest.
- Chargebacks are retrievable via API (listed at https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api/payments/retrieve-chargebacks.md; not read in detail).

## 6. Failed-payment retries

Not documented for subscription plans: no retry count, schedule, grace period, or automatic cancellation. Execution `DECLINED` exists, and card-on-file failures return a `redirect_url`. Plan to build our own dunning (detect DECLINED, message the customer, restrict access) and ask dLocal whether plans retry automatically. **UNCONFIRMED.**

## 7. Local payment methods

Help Center says checkout offers cards, bank transfers and cash, all activated by default and deactivatable per country (https://helpcenter.dlocalgo.com/en/articles/7339131-what-payment-methods-do-i-offer-with-dlocal-go-checkout , https://helpcenter.dlocalgo.com/en/articles/7240008-can-i-activate-or-deactivate-payment-methods-by-country). Colombian methods per dLocal's Colombia page: cards, PSE, Efecty, Bancolombia, Nequi, Mercado Pago, Bre-B, Addi. For subscriptions, only recurring-capable methods (cards, possibly Nequi Push) are usable. Buyers cannot change payment method on an existing subscription: cancel and recreate (https://helpcenter.dlocalgo.com/en/articles/7925887-how-can-my-client-change-the-payment-method-for-a-subscription).

## 8. Fees and settlement

- No fixed or maintenance fees; commission per successful transaction varying by method and country, published only in a PDF we could not read. Refunds under USD 10 cost USD 1. Source: https://helpcenter.dlocalgo.com/en/articles/6960181-dlocal-go-fees
- Colombia commission rate: **UNCONFIRMED** (not on any readable official page; get the PDF or a quote). A third-party comparison site cites 0.99% to 7.99% across all countries; do not rely on it.
- Settlement: cards within 7 days; bank transfer/cash within 3 days (https://helpcenter.dlocalgo.com/en/articles/7240009-how-soon-will-i-see-my-money-in-the-dlocal-go-account). Funds arrive in the company's home-country currency, minus FX.

## 9. Onboarding and approval

- Company must be registered in a supported country (Colombia is on the list). Account setup has step 1 (business info) and step 2 verification; step 2 is required for unlimited transactions and withdrawals. KYC documents, criteria and timing are not published. Sources: https://helpcenter.dlocalgo.com/en/articles/9421580-getting-started-with-dlocal-go , https://helpcenter.dlocalgo.com/en/articles/7229166-what-requirements-do-i-need-to-accept-credit-and-debit-card-payments
- A website is not required to open an account (article title only: https://helpcenter.dlocalgo.com/en/articles/9566823-do-i-need-a-website-to-have-a-dlocal-go-account; body not read).
- Sandbox and live accounts are separate (https://api-sbx.dlocalgo.com, https://api.dlocalgo.com); test cards: https://docs.dlocalgo.com/integration-api/welcome-to-dlocal-go-api.md
- Whether Rita's own legal entity location affects eligibility is **UNCONFIRMED**.

## 10. Constraints on pricing and plan limits

- One plan = one country + currency + frequency + amount. Tiers and limits must be modeled as separate plans; upgrade/downgrade via change-plan charges immediately, no proration.
- `max_periods` caps charges; `day_of_month` anchors billing.
- Editing a plan allows name, description, amount and URLs in the API; the Help Center says frequency is editable too. Effect on existing subscribers is not documented (**UNCONFIRMED**), so price changes should be done by creating a new plan and migrating via change-plan.
- Payer document (CC/NIT) is required for Colombian payments (https://docs.dlocal.com/docs/colombia).
- Usage-based or variable pricing only via card-on-file.

## Questions to put to dLocal Go support

1. Do subscription plans auto-retry failed charges, how often, and what happens after?
2. Can `free_trial_days` be enabled, and how does the first charge behave?
3. Are subscription lifecycle webhooks available beyond `payment_id`?
4. Colombia commission rates for cards and Nequi; are Nequi Push and wallets usable in subscription plans?
5. Does a plan price edit affect active subscribers? Is cancellation reversible?
6. Is SmartFields compatible with recurring in Colombia?
7. Step 2 verification documents and timing for a Colombian company.
