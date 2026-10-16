# Sentinova – Spam Protection

**Product & Technical Specification** · Version 0.1 (Draft)

---

## 1. Overview

**Sentinova – Spam Protection** is a Shopify app that helps merchants stop fake, fraudulent and spam orders. Merchants build **rules** (conditions) and attach **actions** to them. Rules run at two points in the order lifecycle:

1. **Before checkout** – restrict payment options while the customer is still checking out.
2. **After checkout** – act on the order once it has been placed.

The app has three pages: **Dashboard**, **Rules**, and **Orders**.

### 1.1 Problem

Merchants, especially those offering Cash on Delivery (COD), lose money to fake orders, prank orders, repeat chargebacks and bot-driven checkouts. Each bad order costs shipping, packaging, return fees, support time and sometimes a chargeback fee.

### 1.2 Goals

- Let a merchant create flexible, no-code rules in under two minutes.
- Block or flag suspicious orders automatically, with minimal false positives.
- Show clear proof of value: orders stopped and money saved.
- Be fast and safe at checkout: never slow down or break legitimate checkouts.

### 1.3 Non-goals (v1)

- Machine-learning fraud scoring.
- Cross-merchant shared blacklists (possible later).
- Replacing Shopify's own fraud analysis; Sentinova uses it as one signal.

### 1.4 Target users

- **Merchant / store owner** – configures rules and reviews results.
- **Store staff** – reviews flagged orders (access follows Shopify staff permissions).

---

## 2. Core Concepts

| Concept | Description |
| --- | --- |
| **Rule** | A named set of conditions plus the actions to run when they match. |
| **Condition** | One filter (e.g., Order total), one operator (e.g., greater than) and a value. |
| **Action** | Something the app does when a rule matches. Each action belongs to a stage: Before checkout or After checkout. |
| **Stage** | *Before checkout* or *After checkout*. A rule belongs to exactly one stage. |
| **Priority** | The order of rules in the list. Rule 1 is evaluated first. |
| **Match log** | A record of every time a rule matched, and what was done. Feeds the Orders page and Dashboard. |

---

## 3. Rule Filters

Each condition = **Filter + Operator + Value**. The operator list changes depending on the filter type.

| # | Filter | Data type | Operators | Value input |
| --- | --- | --- | --- | --- |
| 1 | **Disposable email** | Boolean + text | *is disposable* / *is not disposable*; email domain *equals*, *contains*, *is in list* | Domain(s), optional |
| 2 | **Phone number validity / VoIP** | Enum | *is* / *is not* → `Invalid`, `VoIP`, `Valid`; phone *starts with*, *equals*, *contains* | Status or number/prefix |
| 3 | **IP address** | Text / network | *equals*, *is in range (CIDR)*, *is in list*, *country is / is not*, *is proxy/VPN/Tor* | IP, CIDR, list or country code |
| 4 | **Customer chargeback count** | Number | *equals*, *greater than*, *greater than or equal*, *less than* | Number, plus time window (30/90/365 days, all time) |
| 5 | **Order total** | Money | *equals*, *greater than*, *less than*, *between* | Amount (store currency) |
| 6 | **Shopify risk level** | Enum | *is*, *is not*, *is any of* → `Low`, `Medium`, `High` | Risk level(s) |
| 7 | **Billing / shipping address mismatch** | Boolean + enum | *is mismatched* / *is matched*; mismatch on *country*, *state/province*, *city*, *ZIP/postal code*, *name* | Which fields to compare |
| 8 | **Shopify Protect eligibility** | Enum | *is* / *is not* → `Eligible`, `Not eligible`, `Pending` | Status |

### 3.1 Combining conditions

- Within a rule, conditions combine with **ALL (AND)** or **ANY (OR)**, chosen by the merchant.
- v1 supports one level (no nested groups). Nested groups are a later enhancement.

### 3.2 Filter availability by stage

Not every signal exists at every moment. Shopify only creates risk data, IP and Protect status once an order exists, and Shopify Functions run in a sandbox. The rule builder must only offer filters that are valid for the chosen stage.

| Filter | Before checkout | After checkout | Notes |
| --- | --- | --- | --- |
| Disposable email | Yes | Yes | Pre-checkout check uses a bundled/synced domain list (no live network call). Live lookup after checkout. |
| Phone invalid / VoIP | Partial | Yes | Pre-checkout: format and country validation only. VoIP detection needs an external lookup, so it runs after checkout. |
| IP address | Limited | Yes | Order's browser IP is reliably available after checkout. Pre-checkout would need a storefront/checkout extension to pass the IP to the app. |
| Chargeback count | Yes | Yes | Pre-computed by the app and stored (e.g., on a customer metafield) so it can be read without a network call. |
| Order total | Yes | Yes | Cart total before checkout; order total after. |
| Shopify risk level | No | Yes | Only exists after the order is created. |
| Billing/shipping mismatch | Limited | Yes | Billing address may not be available during payment steps. |
| Shopify Protect eligibility | No | Yes | Order-level data. |

> **To verify during build:** exact Shopify Functions input fields, plan availability (Plus vs non-Plus) for payment customizations and checkout validation, and which data points are readable at each checkout step. Treat the matrix above as the design intent; the final matrix should be confirmed against current Shopify API docs.

---

## 4. Actions

Actions are selected on the **Actions tab** of the rule editor. The available actions depend on the rule's stage.

### 4.1 Section 1 – Before checkout

| Action | Behavior | Suggested implementation |
| --- | --- | --- |
| **Disable COD** | Hides the Cash on Delivery payment method for the buyer when the rule matches. | Shopify Function – *Payment customization* (hide payment method). |
| **Disable "Pay now" button** | Prevents the buyer from completing the order. Shows a configurable message (e.g., "We couldn't process this order. Please contact support."). | Shopify Function – *Cart & checkout validation* (return a blocking error), with an optional checkout UI extension banner. |

Merchant-configurable options for both: custom buyer-facing message, and an optional internal note.

### 4.2 Section 2 – After checkout

| Action | Behavior | Suggested implementation |
| --- | --- | --- |
| **Cancel & auto refund** | Cancels the order and refunds the payment (if paid). Options: notify customer (on/off), cancel reason (`Fraud` / `Declined` / `Other`). | Admin GraphQL `orderCancel` with refund. |
| **Auto restock** | Returns line-item inventory to stock. Available on its own or as part of Cancel. | `orderCancel` with restock enabled, or inventory adjustment. |
| **Tag order** | Adds a merchant-defined tag (e.g., `sentinova-flagged`) to the order. | `tagsAdd`. |
| **In review** | Puts the order into an "In review" state so it isn't fulfilled until a human decides. | Add tag `sentinova-in-review`, place a fulfillment hold, and show the order in the Orders page with an *In review* status. Merchant can *Approve* (release hold) or *Reject* (cancel & refund) from the app. |

Several After-checkout actions can run together (e.g., Tag + In review, or Tag + Cancel & refund + Restock).

### 4.3 Action validity rules

- A rule can't combine Before-checkout and After-checkout actions.
- *Auto restock* without *Cancel* is allowed only when the order is not yet fulfilled.
- *In review* and *Cancel & refund* conflict; the UI should warn if both are selected.
- Actions on already-fulfilled or already-cancelled orders are skipped and logged as "Skipped – not applicable".

---

## 5. Rule Evaluation Logic

1. Load all **enabled** rules for the stage, sorted by priority (ascending).
2. For each rule, evaluate its conditions (ALL/ANY).
3. On a match, execute the rule's actions and write a **match log**.
4. Continue or stop according to the rule's **"Stop processing further rules"** toggle (default: on).
5. If multiple rules match and processing continues, the **most restrictive** action wins (Cancel > In review > Tag), and all matched rules are logged.

**Reliability requirements**

- Before-checkout evaluation must complete within Shopify's Function time/size limits and must **fail open**: if anything errors, the checkout proceeds normally and the error is logged.
- After-checkout actions are processed asynchronously from webhooks, with retries and idempotency (the same order is never actioned twice).
- If a data source is unavailable (e.g., phone lookup times out), the condition evaluates to *unknown* and does not match. Merchants can choose in settings whether unknown should count as match.

---

## 6. Pages & Screens

### 6.1 Dashboard

**Purpose:** show that the app is working and how much it saves.

**Widgets**

- **Live rules** – count of enabled rules (split Before / After checkout), with a link to Rules.
- **Orders protected** – total orders stopped or acted on.
- **Breakdown by action** – Checkout blocked, COD disabled, Cancelled & refunded, Tagged, In review.
- **Money saved** – total value of orders prevented (see 6.1.1).
- **Trend chart** – orders actioned and money saved over time.
- **Top triggered rules** – which rules fire most often.
- **Date range filter** – Today, 7 days, 30 days, 90 days, custom.
- **Empty state** – onboarding card: *"Create your first rule"*.

#### 6.1.1 "Money saved" definition

- **Cancelled & refunded orders:** order total.
- **Blocked at checkout / COD disabled:** cart total at the time of the block. Because the buyer may have later paid another way, count only blocks where no order was placed by the same checkout within a set window (e.g., 24 hours). Label these as *estimated*.
- **In review:** counted only when the merchant rejects the order; approved orders are excluded.
- **Tag only:** not counted as saved.
- Display in the shop's currency; convert historical values at the order's rate.

### 6.2 Rules Page

**Purpose:** manage all rules.

**List view columns:** priority number (drag handle), rule name, stage (Before / After), conditions summary, actions summary, status toggle (On / Off), times triggered, last triggered, row menu.

**Features**

- **Add rule** button (primary).
- **Edit** and **Delete** (with confirmation).
- **Move up / Move down** (buttons plus drag and drop) to set priority. Priority is per stage.
- **Duplicate** rule.
- **Enable / disable** toggle without deleting.
- Filter by stage and status; search by name.
- Empty state with a short explanation and a *Create rule* button.

#### 6.2.1 Add / Edit Rule (two tabs)

**Tab 1 – Conditions**

- Rule name.
- Stage selector: *Before checkout* / *After checkout* (changes available filters and actions).
- Match type: *All conditions* / *Any condition*.
- Condition rows: **Filter** dropdown → **Operator** dropdown → **Value** input. Add/remove rows.
- Inline validation (e.g., number required for *greater than*; valid CIDR for IP range).

**Tab 2 – Actions**

- Shows the action section for the chosen stage:
  - *Before checkout:* Disable COD, Disable Pay now (+ custom message).
  - *After checkout:* Cancel & auto refund (+ options), Auto restock, Tag order (+ tag name), In review.
- Toggle: *Stop processing further rules*.
- Summary preview: *"If \[conditions\] then \[actions\]"*.

**Save** creates the rule at the bottom of its stage's priority list.

### 6.3 Orders Page

**Purpose:** show every order or checkout the app acted on.

**Table columns:** order number (links to Shopify admin), date, customer (name/email), order total, rule(s) triggered, action(s) taken, status (*Blocked*, *Cancelled & refunded*, *Tagged*, *In review*, *Approved*, *Skipped*, *Failed*), amount saved.

**Features**

- Filter by action, rule, status and date range; search by order number or email.
- Order detail drawer: matched conditions with the actual values (e.g., "Order total 450 > 300"), timeline of actions, errors if any.
- For **In review** orders: *Approve* and *Reject (cancel & refund)* buttons.
- *Undo* for reversible actions (e.g., remove tag, release hold).
- CSV export.
- Pre-checkout blocks (no order exists) appear as rows labelled *Checkout blocked* with the cart total and masked buyer identifiers.

---

## 7. Technical Architecture

### 7.1 Recommended stack

- **App framework:** Shopify app template (React Router / Remix), TypeScript, Node.js.
- **Admin UI:** Polaris web components, embedded via App Bridge.
- **Database:** PostgreSQL with Prisma.
- **Background jobs:** queue worker (e.g., BullMQ + Redis) for webhook processing and retries.
- **Extensions:** Shopify Functions (payment customization, cart & checkout validation) and an optional checkout UI extension.
- **Third-party data:** disposable-email list, phone validation/VoIP lookup API, IP intelligence API (all behind internal adapter interfaces so providers can be swapped).

### 7.2 Components

1. **Embedded admin app** – Dashboard, Rules, Orders.
2. **Rules service** – CRUD, priority ordering, validation.
3. **Evaluation engine** – shared evaluator used by the backend (after checkout) and compiled/serialized for Functions (before checkout).
4. **Enrichment service** – email, phone, IP and chargeback lookups with caching.
5. **Action executor** – calls Shopify Admin GraphQL, idempotent, with retries.
6. **Rule sync** – writes the enabled Before-checkout rules into a shop metafield that Functions read (Functions can't query the app's database).
7. **Analytics aggregator** – pre-computes daily totals for the Dashboard.

### 7.3 Key flows

**Before checkout**

1. Merchant saves/enables a Before-checkout rule → rule sync updates the shop metafield.
2. Buyer reaches payment step → Function reads rules and available data (cart, buyer identity, customer metafields).
3. If a rule matches → hide COD / return blocking error.
4. Function result is reported to the app (via a lightweight extension call or log ingestion) for analytics, where Shopify allows it.

**After checkout**

1. `orders/create` (and `orders/updated`) webhook received → verified (HMAC) and queued.
2. Worker fetches the order, runs enrichment, evaluates After-checkout rules by priority.
3. Action executor applies actions; a match log is saved.
4. Dashboard and Orders page read from the match log.

### 7.4 Shopify access

**Scopes (initial proposal, minimize as the build is finalized)**

- `read_orders`, `write_orders` – read orders, cancel, tag.
- `read_customers` / `write_customers` – chargeback metafield, customer history.
- `read_inventory`, `write_inventory` – restock.
- `read_fulfillments` / `write_fulfillments` (or merchant-managed-fulfillment-orders scopes) – holds for *In review*.
- `read_shopify_payments_disputes` – chargeback counts (Shopify Payments stores only).
- Function-related scopes for payment customization and validation.

**Webhooks:** `orders/create`, `orders/updated`, `orders/cancelled`, `disputes/create`, `disputes/update`, `app/uninstalled`, plus mandatory privacy webhooks `customers/data_request`, `customers/redact`, `shop/redact`.

---

## 8. Data Model (high level)

| Table | Key fields |
| --- | --- |
| **Shop** | id, domain, currency, plan, installed\_at, settings (unknown-as-match, default message, notification email) |
| **Rule** | id, shop\_id, name, stage, match\_type (all/any), priority, enabled, stop\_on\_match, actions (JSON), created\_at, updated\_at |
| **RuleCondition** | id, rule\_id, filter, operator, value (JSON), position |
| **MatchLog** | id, shop\_id, rule\_id, stage, order\_id (nullable), checkout\_ref (nullable), customer\_email\_hash, matched\_conditions (JSON), actions\_taken (JSON), status, amount\_saved, currency, error, created\_at |
| **ChargebackRecord** | id, shop\_id, customer\_id/email\_hash/phone\_hash, dispute\_id, amount, created\_at |
| **DailyStats** | shop\_id, date, rules\_live, orders\_actioned, by\_action (JSON), amount\_saved |
| **ListEntry** (optional) | id, shop\_id, type (email domain / IP / phone), value |

PII (email, phone, IP) is stored hashed or masked wherever the full value isn't needed for display.

---

## 9. Non-Functional Requirements

- **Performance:** Function execution within Shopify limits; admin pages load in under 2 seconds for up to 10,000 log rows (paginated).
- **Reliability:** fail-open at checkout; webhook processing is idempotent with retry and dead-letter handling.
- **Security:** verify webhook HMAC; session-token auth for the admin; encrypt secrets; least-privilege scopes.
- **Privacy & compliance:** implement GDPR/privacy webhooks; define log retention (e.g., 12 months, configurable); don't send more buyer data to third-party APIs than needed.
- **Accessibility & UX:** follow Polaris patterns; responsive in the Shopify admin on mobile.
- **Observability:** structured logging, error alerts, per-provider API health and quota tracking.
- **Localization:** English first; money, dates and phone formats respect the shop locale.

---

## 10. Edge Cases & Risks

| Risk / case | Mitigation |
| --- | --- |
| False positives block real customers | *In review* and *Tag* options; dry-run mode ("Log only"); easy undo; clear buyer message with a contact path. |
| Data not available at checkout (IP, risk level, VoIP) | Stage-aware filters (section 3.2); unknown handling setting. |
| Shopify Function limits (no network, size, time) | Pre-compute data into metafields; keep rule payload compact. |
| Plan restrictions for checkout customizations | Detect plan on install; show which actions are unavailable and why. |
| Third-party API outage or quota | Caching, timeouts, fail-open, provider fallback. |
| Order already paid via gateway that can't auto-refund | Surface a *Failed – manual refund needed* status with instructions. |
| Duplicate webhooks | Idempotency key per order + rule. |
| Conflicting rules | Priority + "stop processing" toggle + most-restrictive-wins. |
| Customer chargeback data limited to Shopify Payments | Also allow merchant-maintained/imported chargeback counts and the app's own records. |
| Legal/consumer-protection concerns when cancelling orders | Merchant-editable cancellation notice; default to notifying the customer. |

---

## 11. Suggested Release Plan

**MVP (v1.0)**

- Rules page with add / edit / delete / reorder / enable.
- Filters: Order total, Disposable email, Shopify risk level, Billing/shipping mismatch, Shopify Protect eligibility, Chargeback count.
- After-checkout actions: Tag, In review, Cancel & refund, Restock.
- Before-checkout actions: Disable COD (payment customization).
- Orders page and a basic Dashboard (live rules, orders actioned, money saved).
- Log-only (dry-run) mode.

**v1.1**

- Phone validity / VoIP and IP address filters (third-party providers).
- Disable "Pay now" via checkout validation.
- Approve / Reject workflow for *In review*, undo, CSV export.
- Trend charts and top-rule analytics.

**v1.2+**

- Block/allow lists, nested condition groups, rule templates (e.g., "COD protection starter pack"), email/Slack alerts, multi-store insights, optional shared reputation network.

---

## 12. Pricing & Billing (to decide)

- Shopify Billing API subscription, with a free tier (limited rules or monthly orders analysed) and paid tiers by order volume or feature set.
- Possible add-on: usage-based charges for phone/IP lookups.

---

## 13. Success Metrics

- Install-to-first-rule activation rate (target: majority within the first session).
- Number of orders actioned and money saved per active store.
- False-positive rate (approved *In review* orders ÷ total *In review*).
- Merchant retention at 30 / 90 days.
- App Store rating and support tickets per 100 installs.

---

## 14. Open Questions

1. Which plans must be supported at launch, and are any Before-checkout actions limited to Shopify Plus?
2. Which providers will supply phone/VoIP and IP intelligence, and who pays for lookups?
3. Should *Disable Pay now* hide all prepaid methods or block order completion entirely (recommended: validation error with a message)?
4. Should the same rule be allowed to span both stages (e.g., Disable COD *and* Tag after order)? Currently: no.
5. Default behavior when multiple rules match: stop on first match, or most-restrictive-wins?
6. Retention period for logs and hashed identifiers.
7. Is a shared/global blocklist across merchants in scope?
