# Research: Channel providers for iMessage and WhatsApp

Resolves GitHub issue #5. Researched 2026-10-14. Terms follow the product terms used in issues #1 and #5 (Channel, Say Hi, Plan, Monday plan); `GLOSSARY.md` is not present on this branch.

## Recommendation

**MVP Channel: WhatsApp Business Platform (Cloud API), with Rita positioned as a task-specific marketing agent for the user's own store, not a general-purpose assistant.**
**Runner-up: iMessage via Sendblue** (Blooio is the cheaper alternative, see below).

Reasons:

1. **Reach in Colombia.** StatCounter shows Android at roughly 83-87% and iOS at roughly 13-17% of Colombian mobile traffic (May-July 2026). An iMessage-only Channel cannot reach about 85% of Colombian users, which defeats a Colombia-first MVP. iMessage providers can fall back to SMS/RCS, but that is not iMessage and weakens the Say Hi step.
2. **Fit with the Wizard.** Say Hi needs the user to send the first inbound message and the wizard page to detect it. On WhatsApp this is a click-to-chat link with a prefilled code, detected via the webhook. The user's message opens a 24h service window in which replies are free (Meta pricing docs).
3. **Business-initiated Monday plan.** Outside the 24h window only approved templates can be sent (charged per message, see Cost). This is a documented mechanism. The iMessage providers either forbid outbound-first on the cheap plans (Blooio Inbound plan, Sendblue AI Agent plan) or require warm-up and prior consent (LoopMessage).
4. **Policy risk is real but manageable, and it is the main open risk.** See below.

## Comparison

| Criterion | WhatsApp Cloud API | Sendblue | Blooio | LoopMessage | Linq |
|---|---|---|---|---|---|
| Inbound first message + detection | Yes, webhook (standard; not re-verified this session) | Yes, "inbound-first" and webhooks on AI Agent plan; none in Sandbox | Yes, signed webhooks, replay | Yes (API says receive and send; webhook details unconfirmed) | Unconfirmed (marketing page only) |
| Business-initiated messages | Only via approved templates outside the 24h window | Not on AI Agent plan ($100/line); full outbound only on Enterprise (custom price) | Not on Inbound plan ($98); Commercial Shared/Dedicated plans allow new contacts (15/day or unlimited) | Possible with dedicated sender with "init conversations", warm-up, and prior opt-in; cold messaging not supported | Unconfirmed |
| Pricing | Per delivered template message by category and country; service messages free in window (see Cost) | Free sandbox (10 verified contacts, shared number); $100/mo per dedicated line; Enterprise custom; no per-message fees | Trial 20 msgs; $39 Starter; $89 Shared; $98 Inbound; $289 Dedicated; $389 Enterprise; no per-message fees | Sandbox $0 (5 contacts); from $59.99/mo (300 daily contacts); from $99.99/mo (1,000); shared sender only for AI assistant use cases | Not on fetched pages |
| Colombia coverage | Native; WhatsApp is dominant in Colombia (share not primary-sourced, see unconfirmed) | Not stated in docs fetched | Not stated; contact sales | "Worldwide", but Apple or carrier restrictions may block delivery; Colombia is not on the excluded-country list | Says "globally", no list |
| Number provisioning | Own number via Meta/BSP onboarding, business verification, display name review (not re-verified this session) | One dedicated number per line, dashboard upgrade | Self-serve pre-warmed iMessage numbers, about 48h restock cadence; custom area code $75 | US any area code, UK; others on request. No Colombian numbers listed | Unconfirmed |
| Policy risk | Clause 4.7 (see below) | Apple grey zone | Apple grey zone; unsolicited outreach violates AUP; Apple-level number restrictions that Blooio cannot reverse | Apple grey zone; explicit no-cold-messaging | Apple grey zone |

## Policy risk

**WhatsApp (Meta).** Clause 4.7 "AI Providers" of Meta's Terms for the WhatsApp Business Platform prohibits AI providers and developers from using the platform to offer general-purpose AI assistants or similar technologies when those are the primary functionality offered, as Meta determines. The page shows a last-updated date of 23 September 2026 and does not state the clause's effective date. Secondary reporting (TechCrunch, Oct 2025) says the change took effect 15 January 2026 and that Meta said it targets assistants distributed by model providers, not businesses using AI for customer support. A company can still use an AI provider as its Solution Provider but may not let Business Platform data train other models.

Assessment: Rita is a vertical, task-scoped marketing agent that acts on one user's own store through approved Plans, not an open-ended chatbot. That is the best-defensible reading, but the test is "primary functionality" as Meta determines, and Rita's whole product is an AI agent. This is a judgment call that cannot be verified from documents; Meta reserves the decision. Mitigations: keep the conversation scope narrow (Plans, approvals, reports), no open-ended Q&A, document the use case in Meta business verification, and keep the Channel behind an adapter so the runner-up can replace it. Regulators in the EU, Italy and Brazil have reportedly opened probes into the rule (secondary source), so terms may change.

**iMessage.** There is no official Apple business API for this use. All four vendors sell access to iMessage through Apple-ID-based infrastructure (a grey zone). Vendor terms confirm the risk: Blooio states policy violations "may lead to Apple-level number restrictions that Blooio cannot reverse". All forbid unsolicited outreach. Apple's own terms were not reviewed in this session.

## Cost (WhatsApp)

- Per-message pricing since 1 July 2025: marketing, utility and authentication templates are charged per delivered message, with rates by template category and recipient country calling code. Service (non-template) messages are free inside the 24h window, and utility templates sent inside it are free.
- A 72h free window applies for conversations started from a Click to WhatsApp ad or Page CTA (not our flow).
- Colombian utility and authentication rates rose on 1 October 2025; COP became a billing currency for new accounts on 1 April 2026. Numeric Colombian rates are only in Meta's COP rate card (CSV or PDF) and were not retrieved. **Unconfirmed: actual per-message COP price of a Monday plan template.**
- Budget implication: one utility template per Project per week is likely cheap, but verify against the rate card before locking pricing.

## Unconfirmed or not verified

- Numeric COP template rates (rate card not fetched).
- WhatsApp inbound webhook, number registration and business verification details (standard Cloud API behavior, not re-fetched).
- WhatsApp share of messaging use in Colombia (no primary source fetched).
- Whether Meta would classify Rita as a "primary functionality" general-purpose assistant. No official guidance found for vertical agents.
- Sendblue, LoopMessage and Linq: webhook payload details, international/Colombian recipient support, and iMessage-to-Android fallback behavior (docs pages fetched were navigation-only or 404). Linq pricing, webhooks, outbound-first and provisioning were not available on the fetched pages.
- Apple's stance on third-party iMessage gateways.
- StatCounter measures web traffic share, not device ownership, and varies monthly (iOS 12.7% May, 15.64% June, 16.72% July 2026).

## Sources

- Meta, Terms for the WhatsApp Business Platform (clause 4.7): https://www.facebook.com/legal/Meta-Terms-for-WhatsApp-Business-Platform
- Meta, WhatsApp pricing: https://developers.facebook.com/docs/whatsapp/pricing
- Meta, pricing updates and COP rate card: https://developers.facebook.com/docs/whatsapp/pricing/updates-to-pricing
- TechCrunch, WhatsApp bars general-purpose chatbots (secondary): https://techcrunch.com/2025/10/18/whatssapp-changes-its-terms-to-bar-general-purpose-chatbots-from-its-platform
- Sendblue pricing: https://www.sendblue.com/pricing
- Sendblue docs: https://docs.sendblue.com
- Blooio: https://blooio.com and https://blooio.com/pricing
- LoopMessage: https://loopmessage.com and https://loopmessage.com/apidocs
- Linq: https://linqapp.com
- StatCounter Colombia mobile OS share: https://gs.statcounter.com/os-market-share/mobile/colombia
