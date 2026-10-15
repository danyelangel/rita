# Rita

Rita is an AI marketing analyst that connects to a business's existing tools, proposes weekly work against a sales goal, waits for approval, executes, and reports. Rita is a Spanish-language product for Colombia: the whole user-facing product is in Spanish, while code, tickets and this glossary are in English. Terms below are the English domain vocabulary; user-facing copy sits behind an i18n layer whose default locale is Spanish.

## Language

**Rita**:
The AI marketing analyst the product delivers; the agent a user talks to.
_Avoid_: Bot, assistant, chatbot

**Project**:
The unit of an account: one brand, one user, one set of connections, one goal. Always one-to-one with its user.
_Avoid_: Account, workspace, brand, tenant

**Wizard**:
The guided onboarding that takes a new user from sign-up to receiving their first plan.
_Avoid_: Onboarding flow, setup

**Company profile**:
The editable, source-cited memory Rita builds about the business from its website and the user's answers.
_Avoid_: Ficha, brand brief, business card

**Key date**:
A dated business event Rita plans around (a sale, a holiday, a launch), with an optional repeat rule (yearly, or every n days, weeks or months). One record per event, kept in the Project's calendar.
_Avoid_: Event, reminder, seasonal date

**Brand guidelines**:
The structured description of how the brand looks and sounds (colour palette, fonts, tone, words to avoid), inferred from an uploaded file or the website and then edited by the user. Part of the Company profile.
_Avoid_: Brand book, style guide

**Goal**:
The monthly sales figure the user wants to reach; every plan and report is expressed against it.
_Avoid_: KPI, target, quota

**Diagnosis**:
The free analysis of a project's history and current setup that Rita runs after connections are made and before the paywall.
_Avoid_: Audit, onboarding report

**Subscription**:
The paid recurring billing relationship of a Project, with its billing events, proration and a due date anchored to the day of the month the user upgraded.
_Avoid_: Plan (a Plan is Rita's work, never billing), tier, membership

**Lapse**:
The state of a Project whose Subscription has ended (cancelled and paid period over, refunded, or payment retries exhausted). It keeps the free Diagnosis and its profile but Rita stops working; re-subscribing restores it.
_Avoid_: Suspension, churn

**Paywall**:
The wizard step after the Diagnosis where the user subscribes to continue.
_Avoid_: Checkout, upgrade screen

## Connections

**CRM & Conversions**:
Shopify, the project's source of truth for customers, orders and conversions.
_Avoid_: Store, ecommerce platform

**Email Marketing**:
The optional connection to Klaviyo for email campaigns and flows.
_Avoid_: Mailing, newsletter tool

**Performance Marketing**:
The optional connection to ad channels (Meta Ads, Google Ads) made through Zernio.
_Avoid_: Paid media, ads

**Lever**:
A connection through which Rita can act, not just read: it holds the permissions to create or change things (email drafts and flows, ad campaigns). A read-only connection is connected but is not a Lever. A Project needs at least one Lever besides Shopify.
_Avoid_: Action channel, write connection

**Channel**:
Where the user talks to Rita outside the web app (a messaging provider), in addition to the always-on web chat.
_Avoid_: Medium, integration

**Pairing**:
The link between a Project and the user's phone number on the Channel. A number is paired to at most one Project at a time; unpairing frees it for another Project. The holder of a paired number can ask Rita which email it is paired to.
_Avoid_: Linking, registering, verifying

**Say Hi**:
The wizard step where the user sends Rita a first message from the number they entered, which completes the Pairing.
_Avoid_: Handshake, verification message

## Work cycle

**Plan**:
The set of Tasks Rita puts forward for a period, each with its reason, cost and effect on the Goal.
_Avoid_: Proposal list, roadmap, campaign

**Task**:
A group of Actions that serve one purpose (for example one promotion: its email, its ads, its budget change). The user can approve every Action in a Task with one click, or Action by Action.
_Avoid_: Item, campaign, workflow

**Action**:
One change Rita proposes to make outside Rita, through one connection (create an email draft, launch an ad, change a budget). Shown to the user as a card, and the same card concept is reused for every kind of change Rita makes. Creatives (ad, email) are reviewed on their Action's card.
_Avoid_: Write action, step, operation

**Creative**:
The text content of one ad or email (copy, subject line, call to action), written by Rita from the Brand guidelines and reviewed and edited on its Action's card. Visuals and layout are not part of it.
_Avoid_: Asset, copy, content

**Plan detail**:
The page for one Plan where the user reviews each Task and its Actions and approves, adjusts or rejects them.
_Avoid_: Approval screen

**Approval**:
The user's explicit yes that lets Rita change anything outside Rita. Nothing is executed without one.
_Avoid_: Confirmation, sign-off

**Pre-approval**:
An Approval given now for an Action that will run at a later time.
_Avoid_: Scheduled approval, auto-approve

**Weekly plan**:
The Plan Rita delivers first thing on the Project's chosen weekday (Tuesday by default), followed by the week's automatic work.
_Avoid_: Monday plan, weekly report, weekly digest

**Nudge**:
A short preprogrammed message Rita sends to get an unanswered Weekly plan approved. Never written by a model.
_Avoid_: Reminder, follow-up

**Pause**:
The state of a Project whose Weekly plan stayed unanswered through every Nudge: Rita stops all work, including generating the next Plan, until the user responds. Not a billing state (see Lapse).
_Avoid_: Suspension, freeze
