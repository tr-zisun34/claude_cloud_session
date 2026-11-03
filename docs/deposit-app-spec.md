# Stagepay: B2B Deposit — Build Specification

**App name:** Stagepay · **App Store listing:** *Stagepay: B2B Deposit* · **Audience:** B2B / wholesale and made-to-order merchants

Version 1.3 (final) · 2026-11-03 · Source documents: *Deposit & Balance App – Requirements* (merchant brief), *Deposit_App_Estimate.xlsx*, *Build_Order_Estimate.xlsx*, *docs/feasibility-test-guide.md*

**Changes in 1.3:** final app name *Stagepay: B2B Deposit*; positioned as a B2B app; all technical identifiers use `stagepay` (metafield namespace, selling plan merchant code, order tags, extension names, order-note prefix) — see section 2 *Naming*.

**Changes in 1.2:** reminder on/off and days moved from Settings to the Dashboard (P1 "Reminders" panel); collection method and days-before-overdue moved from Settings into each rule (P4) and copied onto each order when it is placed.

**Changes in 1.1:** user flows added (section 5); build order and hours aligned with *Build_Order_Estimate.xlsx*; theme block now always renders its own deposit selector (Dawn shows none — feasibility test T10); `read_payment_terms` scope added; feasibility results T1–T3 recorded; untracked Free-plan orders become tracked after upgrade.

This spec turns the merchant requirements into buildable modules. Each module is self-contained:
**Design → Database → Backend → Frontend → Tests → Done criteria**, so one module can be designed, built and tested before the next starts.

App scaffolding (Shopify CLI template, auth, hosting, CI) is **out of scope** of this document and assumed done.

---

## Contents

1. [Product summary](#1-product-summary)
2. [Global conventions](#2-global-conventions)
3. [Data model overview](#3-data-model-overview)
4. [Build order](#4-build-order)
5. [User flows](#5-user-flows)
6. [Module 0 — Foundations & payment feasibility spike](#module-0--foundations--payment-feasibility-spike)
7. [Module 1 — Deposit rules & selling plan sync](#module-1--deposit-rules--selling-plan-sync)
8. [Module 2 — Order sync & payment status engine](#module-2--order-sync--payment-status-engine)
9. [Module 3 — Storefront (theme block & validation)](#module-3--storefront-theme-block--validation)
10. [Module 4 — Balance collection](#module-4--balance-collection)
11. [Module 5 — Emails & reminders](#module-5--emails--reminders)
12. [Module 6 — Dashboard & order detail](#module-6--dashboard--order-detail)
13. [Module 7 — Settings, onboarding & plan detection](#module-7--settings-onboarding--plan-detection)
14. [Module 8 — App billing (Free / Pro)](#module-8--app-billing-free--pro)
15. [Module 9 — Compliance, security & audit log](#module-9--compliance-security--audit-log)
16. [Module 10 — End-to-end QA & launch](#module-10--end-to-end-qa--launch)
17. [Appendix A — Full Prisma schema](#appendix-a--full-prisma-schema)
18. [Appendix B — Shopify scopes, webhooks & APIs](#appendix-b--shopify-scopes-webhooks--apis)
19. [Appendix C — Later phases (not in v1)](#appendix-c--later-phases-not-in-v1)

---

## 1. Product summary

**Stagepay: B2B Deposit** is a public Shopify app for **B2B, wholesale and made-to-order merchants**. It lets them **take a deposit at checkout and collect the remaining balance later, on one Shopify order**, on every Shopify plan.

**Core features:** deposit rules per product/collection · deposit at checkout with one *Partially paid* order · balance collection by saved card, payment link or recorded bank transfer · automatic reminders and overdue alerts · dashboard of deposits held, balances owed and overdue orders.

**Typical buyers:** restaurants, hotels, offices, retailers and contractors placing large, made-to-order or bulk orders with the merchant.

**v1 scope:** one deposit + one balance payment (e.g. 30% / 70%). The data model supports N payment steps so that 3–4 step schedules (Appendix C) need no schema change.

**Example used throughout:** AB Furniture sells 20 tables for **$10,000**. Rule = 30% deposit. Buyer pays **$3,000** at checkout. Shopify creates **one order** (*Partially paid*, $7,000 outstanding). When the tables are ready, staff click **Collect balance** → app charges the saved card or emails a payment link → order becomes **Paid**.

### Principles

| # | Principle |
|---|---|
| P1 | **Shopify is the source of truth for money.** Paid/outstanding amounts are always read from the Shopify order; the app never calculates them independently. |
| P2 | **The app is the source of truth for the schedule** (steps, triggers, reminders, statuses). |
| P3 | **Never touch card data.** All payments go through Shopify checkout, saved-card mandates or manual payment records. |
| P4 | **One order per sale.** No flow may create a second Shopify order for a payment. |
| P5 | **Every money action is idempotent and logged.** |

---

## 2. Global conventions

| Area | Convention |
|---|---|
| Stack | Node.js + TypeScript, React Router (Shopify app template), Polaris web components / Polaris React, Prisma + PostgreSQL |
| Shopify API | Admin GraphQL API, pinned version (`2026-10` or later). All calls through one `shopifyAdmin(shop)` client wrapper with retry on throttling (HTTP 429 / `THROTTLED`). |
| Money | `Decimal(12,2)` in DB + `currencyCode` (ISO 4217). Never floats. Shopify `MoneyV2` amounts are parsed with a decimal library. |
| IDs | DB primary keys: `cuid()`. Shopify IDs stored as full GIDs (`gid://shopify/Order/123`). |
| Multi-tenancy | Every table has `shopId`; every query filters by the session's shop. |
| Jobs | **pg-boss** (Postgres-backed queue). Job names `module.action` (e.g. `balance.charge`). Retries: 3 with exponential backoff. |
| Time | Store UTC. Show in the shop's IANA timezone (`shop.ianaTimezone`). |
| Errors | Typed `AppError(code, message, meta)`. UI shows friendly text; logs keep `code` + `meta`. |
| Logging | Structured JSON logs (`shop`, `orderGid`, `jobId`, `code`). Error tracking (Sentry or similar). |
| Feature flags | `ShopSettings.features` JSON for plan-gated or beta features. |
| Naming | Files `kebab-case`, services `<domain>.server.ts`, routes per React Router file conventions. |
| Testing | Vitest (unit + integration with a test Postgres), Playwright (admin UI + storefront E2E on a development store), recorded GraphQL fixtures for Shopify responses. |

### Naming

| Item | Name |
|---|---|
| App / brand | Stagepay (listing: *Stagepay: B2B Deposit*) |
| Metafield namespace (app-owned) | `stagepay` (keys: `policy`, `policies`, `plan_ids`, `minimums`, `schedule`) |
| Selling plan `merchantCode` | `stagepay-<ruleId>` |
| Order tags | `stagepay-deposit-paid`, `stagepay-balance-due`, `stagepay-overdue`, `stagepay-paid` |
| Order note prefix | `[Stagepay]` |
| Theme extension / block | `stagepay-block` / `stagepay-deposit` ("Stagepay deposit") |
| Validation Function | `stagepay-min-order` |
| Email From | `"<Shop name>" <notifications@mail.stagepay.app>` |

### Shared status enums

```
DepositOrderStatus: DEPOSIT_PAID | BALANCE_REQUESTED | PAYMENT_FAILED | OVERDUE | PAID | CANCELLED
PaymentStepStatus:  WAITING | REQUESTED | PROCESSING | PAID | FAILED | CANCELLED
StepTrigger:        AT_CHECKOUT | MANUAL | ON_FULFILLMENT | DAYS_AFTER_ORDER
CollectionMethod:   SAVED_CARD | PAYMENT_LINK | MANUAL_RECORD
```

---

## 3. Data model overview

```
Shop 1───1 ShopSettings
 │ 1───1 BillingState
 │
 ├──* DepositRule ──* RuleTarget (product / collection)
 │        └── sellingPlanGroupGid (Shopify)
 │
 ├──* DepositOrder ──* PaymentStep ──* PaymentAttempt
 │        │                └──* Reminder
 │        └──* EmailLog
 │
 ├──* EmailTemplate
 ├──* AuditLog
 └──* WebhookEvent (idempotency)
```

Each module below defines the tables it owns. The combined schema is in [Appendix A](#appendix-a--full-prisma-schema).

---

## 4. Build order

Build in the order money flows: rule → product page → order → collect balance → emails → polish. Each step can be tested with real test orders before the next starts.

| Step | Build | Pages / extensions | Spec module | Est. hours* |
|---|---|---|---|---|
| 0 | Foundations: shop record, Shopify client, jobs, webhook receiver | — | 0 | 14 |
| 1 | Deposit rules list + create/edit rule (incl. collection method, days before overdue), selling plan sync | P3, P4 | 1 | 20 |
| 2 | Theme app extension: own deposit selector + amounts + policy | E1 | 3 | 13 |
| 3 | Order sync & status engine | — | 2 | 20 |
| 4 | Dashboard, simple version (orders list) | P1 (part) | 6 | 5 |
| 5 | Order detail + Collect balance / Record payment | P2 | 4, 6 | 28 |
| 6 | Emails with default wording + reminders | — | 5 | 13 |
| 7 | Settings (branding, emails, policy) + plan detection | P5 | 7 | 3 |
| 8 | Email templates editor | P6 | 5 | 3 |
| 9 | Dashboard, full version: totals, Reminders panel, setup checklist | P1 (finish) | 5, 6, 7 | 7 |
| 10 | Plan & billing + usage limit | P7 | 8 | 5 |
| 11 | Checkout validation Function (minimum order value) | E2 | 3 | 4 |
| 12 | Privacy webhooks, uninstall cleanup, activity log | P2 section | 9 | 7 |
| 13 | Full testing and launch | — | 10 | 29 |
| | **Base total** | | | **171** |

\*From *Build_Order_Estimate.xlsx* (base hours; +20% buffer = 205 hours ≈ 5 weeks for one developer).

**Milestones:** after step 2 buyers can pay a deposit · after step 5 the app works end to end (demo-able) · after step 6 no manual chasing · after step 10 ready to sell · after step 13 ready for App Store review.

### Pages and extensions (v1)

| Code | Page / extension | Route / location |
|---|---|---|
| P1 | Dashboard (home) | `/app` |
| P2 | Order detail (+ Collect balance, Record payment pop-ups) | `/app/orders/:id` |
| P3 | Deposit rules list | `/app/rules` |
| P4 | Create / edit rule | `/app/rules/new`, `/app/rules/:id` |
| P5 | Settings | `/app/settings` |
| P6 | Email templates (list + editor) | `/app/settings/emails`, `/app/settings/emails/:key` |
| P7 | Plan & billing | `/app/billing` |
| E1 | Theme app extension "Stagepay deposit" (product page) | `extensions/stagepay-block` |
| E2 | Cart & checkout validation Function (minimum order) | `extensions/stagepay-min-order` |
| BG | Background: webhooks, jobs, emails (no UI) | server |

---

## 5. User flows

Each step names who acts and where (page, extension, BG = background, Shopify = Shopify's own checkout/admin). Example order: 20 tables, $10,000, 30% deposit.

### Flow 1 — Install and first setup
1. Merchant installs from the App Store and approves scopes (Shopify).
2. BG saves the shop (name, currency, timezone, plan), creates default settings, the 7 default emails and the Free plan; sets plan-based features.
3. P1 shows the setup checklist: ① create a rule ② add the block to the product page ③ place a test order ④ collect a test balance.

### Flow 2 — Create a deposit rule
1. P3 → **Create rule**.
2. P4: name, 30%, products/collection, minimum order $5,000, balance trigger, **collection method (saved card / payment link)**, **days before overdue (7)**, buyer label, policy; live preview "$10,000 → $3,000 today, $7,000 later".
3. P4 validates (targets, 1–99%, no product in two active rules) → **Save**.
4. BG creates the selling plan group in Shopify and attaches products; writes policy metafield (for E1) and minimums metafield (for E2).
5. P3 shows **Active · Synced** (or **Sync failed · Retry**).

### Flow 3 — Edit, turn off or delete a rule
- Edit (P4): Shopify plan updated; existing orders keep their schedule.
- Turn off (P3 switch) / Delete (P4): selling plan group deleted; existing orders unaffected.

### Flow 4 — Add the block to the product page (once)
1. P1 checklist ② → deep link opens the theme editor with E1 added.
2. Merchant adjusts block settings → **Save**. Required because Dawn renders no selling-plan picker (feasibility T10).

### Flow 5 — Buyer places a deposit order
1. Product page + E1: ○ Pay in full · ● Pay 30% deposit — $300 today, $700 later · policy.
2. Buyer picks deposit, quantity 10 → E1 shows $3,000 today, $7,000 later → Add to cart (cart line shows the plan name).
3. Shopify checkout + E2: $10,000 ≥ $5,000 → allowed (below minimum → "Deposit option requires an order of at least $5,000", checkout blocked).
4. Shopify checkout: due today $3,000 / due later $7,000; buyer pays; card vaulted.
5. Shopify creates ONE order, *Partially paid*, $7,000 outstanding; sends its order confirmation.

### Flow 6 — App picks up the order
1. Shopify webhook `orders/create` → BG.
2. BG detects our selling plan; saves total/paid/outstanding, mandate id, payment link.
3. BG copies the rule's collection method and days before overdue onto the order (later rule edits don't change it).
4. BG creates steps: Deposit $3,000 PAID · Balance $7,000 WAITING; tags `stagepay-deposit-paid`, `stagepay-balance-due`; note line; schedules DAYS_AFTER_ORDER trigger if set; increments Free-plan usage.
5. P1 lists it under **Balance due**.

### Flow 7 — Start the balance request (3 triggers)
- **A. Staff button:** P1 → P2 → **Collect balance** → pop-up "Collect $7,000 from Restaurant ABC" · method pre-selected from the rule (saved card •••• 4242 or payment link), staff may switch → Confirm.
- **B. On shipping:** staff mark the order fulfilled in Shopify → webhook → BG uses the order's collection method (from its rule).
- **C. Days after order:** scheduled BG job.

BG first re-syncs the order (already paid → stop) and locks the step (double click → no-op), then continues with Flow 8 or 9.

### Flow 8 — Collect by saved card
1. BG creates a PaymentAttempt with a unique idempotency key and calls `orderCreateMandatePayment` for $7,000.
2. BG polls `orderPaymentStatus` every 15 s.
3. Success → same order *Paid*; step PAID; tags `stagepay-paid`; note line; reminders cancelled; buyer email "Payment received"; P2 toast "Balance paid".
4. Failure → Flow 12.

### Flow 9 — Collect by payment link
1. BG reads `additionalPaymentCollectionUrl`, emails buyer "Your order is ready — $7,000 due" with **Pay now**.
2. Step REQUESTED; due date = today + the order's days before overdue (from its rule); reminders scheduled (Flow 11).
3. Buyer pays on Shopify's payment page → same order *Paid*.
4. Webhook → BG: step PAID, tags/note updated, reminders cancelled, "Fully paid" email; P1 moves it to **Paid**.

### Flow 10 — Record a bank transfer
1. Buyer wires money.
2. P2 → **Record payment**: amount, method, date, reference → Record.
3. BG validates amount ≤ outstanding and calls `orderCreateManualPayment` on the same order.
4. Full amount → *Paid*, reminders stop, "Fully paid" email. Part amount → stays *Partially paid*, reminders continue.
5. Activity log: "Sarah recorded $7,000 bank transfer".

### Flow 11 — Reminders and overdue
1. Days 3, 7, 14 after request (set in the P1 **Reminders** panel; skipped entirely if reminders are off) → BG sends reminder with Pay now link (skipped if paid/cancelled).
2. Due date passed and unpaid → daily BG check sets OVERDUE, tag `stagepay-overdue`, staff alert email.
3. P1 Overdue tab; P2 red banner "Do not ship: $7,000 unpaid"; staff can resend link, charge again or record payment.

### Flow 12 — Failed card payment
1. Charge fails → step FAILED, order PAYMENT_FAILED.
2. Buyer email "Action needed" with payment link; staff alert "Card charge failed".
3. P2 shows the error; staff retry or record payment; buyer paying by link continues as Flow 9 steps 3–4.

### Flow 13 — Order changes
| Change | Where | BG result |
|---|---|---|
| Edited (20 → 25 tables) | Shopify admin | Re-sync: total $12,500, owed $9,500, step amount updated |
| Discount after deposit | Shopify admin | Owed updated |
| Refund | Shopify admin | Paid updated |
| Cancelled | Shopify admin | CANCELLED, steps cancelled, reminders stopped |
| Cancel balance | P2 | Requests and reminders stop; Shopify order unchanged |

### Flow 14 — Settings and email templates
1. P5: logo, reply-to, staff alert email, default policy → Save (BG updates policy metafield for E1).
   Reminders are changed on P1 (Reminders panel); collection method and days before overdue are set per rule on P4.
2. P5 → P6: choose an email, edit subject/body with placeholders, preview, send test, save or reset.

### Flow 15 — App billing (Free → Pro)
1. P1 banner "7 of 10 free deposit orders used this month".
2. 11th order: deposit still taken in Shopify; app marks it **untracked** (greyed, no actions/reminders).
3. P7 → **Upgrade to Pro ($29/month)** → Shopify approval page (test charge on dev stores).
4. `app_subscriptions/update` → plan PRO; **untracked orders of the current period become tracked** (re-sync, reminders scheduled if requested).
5. Monthly BG job resets Free usage.

### Flow 16 — Uninstall and privacy
| Event | BG result |
|---|---|
| Uninstall | Stop jobs; purge shop data after 48 h (cancelled on reinstall). The app loses API access at uninstall, so it cannot delete its selling plans then — ⚠ verify whether Shopify removes app-created selling plan groups automatically (feasibility guide). |
| `customers/data_request` | Export that customer's records to the shop owner |
| `customers/redact` | Anonymise name/email |
| `shop/redact` | Delete all shop data |

### Flow → page map
| Part | Flows |
|---|---|
| P1 Dashboard (incl. Reminders panel) | 1, 4, 6, 7, 9, 11, 15 |
| P2 Order detail | 7, 8, 10, 11, 12, 13 |
| P3 Rules / P4 Create-edit rule | 2, 3, 7, 9 (method and overdue days) |
| P5 Settings | 14 |
| P6 Email templates | 14 |
| P7 Plan & billing | 15 |
| E1 Theme extension | 4, 5 |
| E2 Validation Function | 2 (minimums saved), 5 (checkout blocked) |
| BG | all |

---

## Module 0 — Foundations & payment feasibility spike

### Purpose
Shared building blocks used by every module, and a **go/no-go test** of the Shopify payment capabilities the whole app depends on.

### 0.1 Design
No merchant-facing UI. A hidden `/app/dev` page (only when `NODE_ENV !== 'production'`) lists spike results and lets the developer run jobs manually ("Run reminders now", "Trigger balance for order").

### 0.2 Database

```prisma
model Shop {
  id            String   @id @default(cuid())
  shopDomain    String   @unique        // ab-furniture.myshopify.com
  shopGid       String   @unique
  name          String
  email         String
  currencyCode  String
  ianaTimezone  String
  planName      String                  // e.g. "Grow", "Shopify Plus"
  isPlus        Boolean  @default(false)
  isDevStore    Boolean  @default(false)
  installedAt   DateTime @default(now())
  uninstalledAt DateTime?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}

model WebhookEvent {
  id          String   @id @default(cuid())
  shopId      String
  webhookId   String   @unique          // X-Shopify-Webhook-Id header
  topic       String
  receivedAt  DateTime @default(now())
  processedAt DateTime?
  error       String?
  @@index([shopId, topic])
}
```

### 0.3 Backend
| Component | Responsibility |
|---|---|
| `shopify-admin.server.ts` | GraphQL client per shop; throttling retry; typed query helper `gql<T>()`. |
| `queue.server.ts` | pg-boss start/stop, `enqueue(name, data, opts)`, `schedule(name, cron)`. |
| `money.server.ts` | `parseMoney(MoneyV2)`, `formatMoney(amount, currency, locale)`, `percentOf(total, pct)` (rounds half-up to 2 dp). |
| `webhook-guard.server.ts` | Verifies HMAC (via template), inserts `WebhookEvent` by `webhookId`; if duplicate → returns 200 and skips. |
| `shop.server.ts` | On install/auth: upsert `Shop` from `shop { id name email currencyCode ianaTimezone plan { displayName partnerDevelopment shopifyPlus } }`. |

### 0.4 Payment feasibility spike (run first, record results)

Run on **two development stores**: one non-Plus, one Plus. Use Shopify Payments test mode (`4242 4242 4242 4242`) or the Bogus Gateway.

| # | Question | How to test | Result needed |
|---|---|---|---|
| S1 | Does a selling-plan deposit create ONE partly-paid order? | Create selling plan (30% checkout charge), buy product | Order `displayFinancialStatus = PARTIALLY_PAID`, `totalOutstandingSet = 70%` |
| S2 | Is a vaulted payment method available on that order? | Query `paymentCollectionDetails.vaultedPaymentMethods` | ≥ 1 mandate id |
| S3 | Can the app charge the **whole** remaining balance? | `orderCreateMandatePayment` without `amount` | Order becomes `PAID` |
| S4 | Can the app charge **part** of the balance? | `orderCreateMandatePayment` with `amount` | Record per plan (needed for Appendix C) |
| S5 | Does the "additional payment" link pay on the same order? | Open `paymentCollectionDetails.additionalPaymentCollectionUrl` in a private window, pay | No new order; same order `PAID` |
| S6 | Does a manual payment of any amount attach to the order? | `orderCreateManualPayment` with `amount` | Outstanding reduced by that amount |
| S7 | Which scopes need Shopify approval? | Request scopes in app config | List + approval lead time |

**Fallback decisions**
- S3 fails on non-Plus → v1 uses **payment link only** on non-Plus (`ShopSettings.features.savedCardCharge = false`).
- S5 fails → payment link uses Shopify's `orderInvoiceSend` email instead of the app's own email link.
- S1 fails → **stop and re-plan** (core assumption broken).

**Results so far (non-Plus dev store, Bogus Gateway, 2026-11-03)**

| Test | Result |
|---|---|
| T1 / S1 setup — `sellingPlanGroupCreate` (30%, `TIME_AFTER_CHECKOUT`, `P90D`, `PRE_ORDER`) | ✅ Pass, product attached |
| T10 — Dawn shows the deposit option | ⚠ Storefront exposes `selling_plan_groups`, but Dawn renders **no picker** → E1 must render its own selector |
| T2 / S1 — deposit checkout | ✅ ONE order `PARTIALLY_PAID`, received 3,000, outstanding 7,000, line item carries the selling plan |
| T3 / S2 — vaulted card | ✅ `PaymentMandate` present |
| S5 prerequisite — balance link | ✅ `additionalPaymentCollectionUrl` present (payment not yet tested) |
| Scopes | `paymentTerms` needs **`read_payment_terms`** (added to Appendix B) |
| Pending | T4 (whole-balance charge), T5, T6, T7, T8, T9, T11, uninstall behaviour of selling plans; all on Plus store |

### 0.5 Tests
- Unit: `money.server.ts` rounding (30% of $10,000.01 = $3,000.00 → balance $7,000.01), currency formatting.
- Unit: webhook guard rejects duplicate `webhookId`.
- Integration: queue enqueue → handler runs → retries 3 times on throw.

### 0.6 Done criteria
- [ ] Spike table S1–S7 filled in and shared with the merchant.
- [ ] Client, queue, money and webhook guard merged with tests passing.

---

## Module 1 — Deposit rules & selling plan sync

### Purpose
Merchants create deposit rules; the app mirrors each active rule as a Shopify **selling plan group** attached to the chosen products/collections, so buyers see a "Pay deposit" purchase option.

**Covers:** FR-1 to FR-6.

### 1.1 Design

**Screen: Deposit rules (list)** — `/app/rules`
| Element | Detail |
|---|---|
| Page title | "Deposit rules" + primary button **Create rule** |
| Empty state | Illustration, "Take deposits on made-to-order products", button **Create your first rule** |
| Table columns | Name · Deposit (e.g. "30%" / "$500") · Applies to (e.g. "Custom Tables collection") · Balance due (e.g. "When staff request it") · Status toggle (Active / Off) · ⋯ (Edit, Delete) |
| Sync badge | "Synced" / "Syncing…" / "Sync failed — Retry" per row |

**Screen: Create / edit rule** — `/app/rules/new`, `/app/rules/:id`
| Field | Type | Validation |
|---|---|---|
| Rule name | text | required, ≤ 60 chars, unique per shop |
| Deposit type | radio: Percentage / Fixed amount | required |
| Deposit value | number | % : 1–99 · fixed: > 0, ≤ 1,000,000 |
| Applies to | resource picker: products and/or collections | ≥ 1 item |
| Minimum order value | money (optional) | ≥ 0 |
| Balance due | select: *When staff request it* / *When the order is shipped* / *X days after order* | X: 1–365 |
| Final due date (days after order) | number | 1–365, default 90. Used as the Shopify payment-terms due date. |
| How to collect the balance | radio: *Charge saved card* / *Email payment link* | required, default *Charge saved card*; card option disabled with tooltip if the store's plan can't charge saved cards |
| Days before overdue | number | 1–60, default 7. Counted from the day the balance is requested. |
| Label shown to buyers | text | default "Pay {deposit} deposit now, rest later"; ≤ 80 chars |
| Deposit policy | textarea | ≤ 500 chars |
| Preview box | read-only | "On a $10,000 order: $3,000 today, $7,000 later" (live) |

Buttons: **Save** (primary), **Cancel**, **Delete** (edit only, destructive, confirm modal: "Existing orders keep their schedule").

**Rule conflicts:** a product may belong to only one active rule. On save, if overlap → inline error listing the conflicting products and rule.

### 1.2 Database

```prisma
model DepositRule {
  id                   String   @id @default(cuid())
  shopId               String
  name                 String
  depositType          DepositType          // PERCENTAGE | FIXED
  depositValue         Decimal  @db.Decimal(12,2)
  minOrderValue        Decimal? @db.Decimal(12,2)
  balanceTrigger       StepTrigger          // MANUAL | ON_FULFILLMENT | DAYS_AFTER_ORDER
  balanceTriggerDays   Int?
  finalDueDays         Int      @default(90)
  collectionMethod     CollectionMethod @default(SAVED_CARD)   // SAVED_CARD | PAYMENT_LINK
  overdueDays          Int      @default(7)
  buyerLabel           String
  policyText           String?
  active               Boolean  @default(true)
  sellingPlanGroupGid  String?
  sellingPlanGid       String?
  syncStatus           SyncStatus @default(PENDING)   // PENDING | SYNCED | FAILED
  syncError            String?
  deletedAt            DateTime?
  createdAt            DateTime @default(now())
  updatedAt            DateTime @updatedAt
  targets              RuleTarget[]
  @@unique([shopId, name])
  @@index([shopId, active])
}

model RuleTarget {
  id           String     @id @default(cuid())
  ruleId       String
  targetType   TargetType // PRODUCT | COLLECTION
  targetGid    String
  title        String     // cached for display
  rule         DepositRule @relation(fields: [ruleId], references: [id], onDelete: Cascade)
  @@unique([ruleId, targetGid])
}

enum DepositType { PERCENTAGE FIXED }
enum TargetType  { PRODUCT COLLECTION }
enum SyncStatus  { PENDING SYNCED FAILED }
```

### 1.3 Backend

**Routes (loaders/actions)**
| Route | Method | Action |
|---|---|---|
| `/app/rules` | GET | list rules (not deleted) |
| `/app/rules` | POST `intent=toggle` | set `active`, enqueue `rules.sync` |
| `/app/rules/new`, `/app/rules/:id` | POST `intent=save` | validate (zod), upsert rule + targets, enqueue `rules.sync` |
| `/app/rules/:id` | POST `intent=delete` | soft delete, enqueue `rules.sync` |

**Service `rules.server.ts`**
- `validateRule(input)` — zod schema + overlap check: expand collections to product GIDs (`collection.products`, paginated) and compare with other active rules.
- `syncRule(ruleId)` (job `rules.sync`):
  1. **Active & no group** → `sellingPlanGroupCreate`.
  2. **Active & group exists** → `sellingPlanGroupUpdate` (plan fields) + `sellingPlanGroupAddProducts` / `sellingPlanGroupRemoveProducts` for target diff.
  3. **Inactive or deleted** → `sellingPlanGroupDelete` (existing orders are unaffected; Shopify keeps the plan snapshot on orders).
  4. Set `syncStatus`; on failure store `syncError`, retry 3×.

**Selling plan mapping** (⚠ verify enum values against the pinned API version during spike S1)

```graphql
mutation CreateDepositPlan($input: SellingPlanGroupInput!, $resources: SellingPlanGroupResourceInput) {
  sellingPlanGroupCreate(input: $input, resources: $resources) {
    sellingPlanGroup { id sellingPlans(first: 1) { nodes { id } } }
    userErrors { field message code }
  }
}
```

```jsonc
{
  "input": {
    "name": "<rule.buyerLabel>",
    "merchantCode": "stagepay-<rule.id>",
    "options": ["Deposit"],
    "sellingPlansToCreate": [{
      "name": "<rule.buyerLabel>",
      "options": "<e.g. 30% deposit>",
      "category": "PRE_ORDER",
      "billingPolicy": { "fixed": {
        "checkoutCharge": { "type": "PERCENTAGE", "value": { "percentage": 30 } },
        // FIXED rule → { "type": "PRICE", "value": { "fixedValue": "500.00" } }
        "remainingBalanceChargeTrigger": "TIME_AFTER_CHECKOUT",
        "remainingBalanceChargeTimeAfterCheckout": "P90D"      // rule.finalDueDays
      }},
      "deliveryPolicy":  { "fixed": { "fulfillmentTrigger": "UNKNOWN" } },
      "inventoryPolicy": { "reserve": "ON_SALE" }
    }]
  },
  "resources": { "productIds": ["gid://shopify/Product/1"] }
}
```

Notes:
- **Fixed-amount deposits** are per unit in Shopify selling plans. UI must say "$500 per item". (Order-level fixed deposits → Appendix C.)
- `finalDueDays` only sets the *latest* due date; the app collects earlier according to `balanceTrigger`.
- Collections are expanded to product IDs at sync time; a daily job `rules.resyncCollections` adds products newly added to targeted collections.

### 1.4 Frontend
| File | Content |
|---|---|
| `app.rules._index.tsx` | Polaris IndexTable, empty state, toggle, sync badge with Retry |
| `app.rules.$id.tsx` | Form, ResourcePicker (`shopify.resourcePicker({type:'product'|'collection', multiple:true})`), live preview, save bar |
| `components/DepositPreview.tsx` | Pure component: `(type, value, sampleTotal) → {today, later}` |

### 1.5 Tests
| Type | Case |
|---|---|
| Unit | zod validation: 0%, 100%, negative, missing targets → errors |
| Unit | Preview: 30% of $10,000 → $3,000 / $7,000; $500 fixed × 20 items → $10,000 / $0 shows warning |
| Unit | Overlap detection product-in-collection vs product |
| Integration | Save rule → job → `sellingPlanGroupCreate` called with expected variables (GraphQL mock) |
| Integration | Toggle off → `sellingPlanGroupDelete`; edit targets → add/remove products diff |
| Integration | Edit rule method/overdue days → existing orders keep their copied values; new orders use the new ones |
| Integration | Shopify `userErrors` → `syncStatus = FAILED`, message shown |
| Manual (dev store) | Product page shows "Pay 30% deposit" option after save |

### 1.6 Done criteria
- [ ] Merchant can create, edit, toggle, delete rules.
- [ ] Each active rule has a synced selling plan visible on the product in Shopify admin.
- [ ] Overlaps blocked; sync failures visible and retryable.

---

## Module 2 — Order sync & payment status engine

### Purpose
Detect deposit orders, create their payment schedule, and keep paid/outstanding amounts and statuses in sync with Shopify.

**Covers:** FR-10, FR-11, FR-12, FR-15, FR-17, FR-19 (tag), FR-20 (note), edge cases (edits, refunds, cancellation, duplicate webhooks).

### 2.1 Design
No new screen (data shown in Module 6). Defines the **state machines**:

**DepositOrder**
```
             (order created with deposit plan)
                        │
                        ▼
                  DEPOSIT_PAID ──(balance requested)──► BALANCE_REQUESTED
                        │                                  │      │
                        │                    (charge fails)│      │(due date passed)
                        │                                  ▼      ▼
                        │                          PAYMENT_FAILED  OVERDUE
                        │                                  │      │
                        └──────(outstanding = 0)───────────┴──────┴──► PAID
any state ──(order cancelled)──► CANCELLED
```

**PaymentStep:** `WAITING → REQUESTED → PROCESSING → PAID` · `PROCESSING → FAILED → REQUESTED (retry/link)` · any → `CANCELLED`.

### 2.2 Database

```prisma
model DepositOrder {
  id                 String   @id @default(cuid())
  shopId             String
  orderGid           String
  orderName          String               // "#1052"
  ruleId             String?
  customerGid        String?
  customerName       String?
  customerEmail      String?
  currencyCode       String
  totalAmount        Decimal  @db.Decimal(12,2)   // currentTotalPriceSet
  paidAmount         Decimal  @db.Decimal(12,2)   // totalReceivedSet - refunds
  outstandingAmount  Decimal  @db.Decimal(12,2)   // totalOutstandingSet
  status             DepositOrderStatus
  balanceDueAt       DateTime?            // requestedAt + overdueDays
  finalDueAt         DateTime             // from payment terms / rule.finalDueDays
  mandateGid         String?              // vaulted payment method id
  collectionMethod   CollectionMethod     // copied from rule at creation
  overdueDays        Int                  // copied from rule at creation
  paymentLinkUrl     String?
  fulfilledAt        DateTime?
  cancelledAt        DateTime?
  lastSyncedAt       DateTime
  createdAt          DateTime @default(now())
  updatedAt          DateTime @updatedAt
  steps              PaymentStep[]
  @@unique([shopId, orderGid])
  @@index([shopId, status])
  @@index([shopId, balanceDueAt])
}

model PaymentStep {
  id            String   @id @default(cuid())
  depositOrderId String
  position      Int                    // 1 = deposit, 2 = balance (N later)
  label         String                 // "Deposit", "Balance"
  amount        Decimal  @db.Decimal(12,2)
  trigger       StepTrigger
  triggerDays   Int?
  status        PaymentStepStatus
  method        CollectionMethod?
  requestedAt   DateTime?
  paidAt        DateTime?
  transactionGids String[]             // Shopify transactions matched to this step
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  depositOrder  DepositOrder @relation(fields: [depositOrderId], references: [id], onDelete: Cascade)
  attempts      PaymentAttempt[]
  @@unique([depositOrderId, position])
}

enum DepositOrderStatus { DEPOSIT_PAID BALANCE_REQUESTED PAYMENT_FAILED OVERDUE PAID CANCELLED }
enum PaymentStepStatus  { WAITING REQUESTED PROCESSING PAID FAILED CANCELLED }
enum StepTrigger        { AT_CHECKOUT MANUAL ON_FULFILLMENT DAYS_AFTER_ORDER }
enum CollectionMethod   { SAVED_CARD PAYMENT_LINK MANUAL_RECORD }
```

### 2.3 Backend

**Webhooks** (all go through `webhook-guard`, then enqueue `orders.sync { orderGid }` — handlers return 200 within 1 s)
| Topic | Handling |
|---|---|
| `orders/create` | If any line item has a selling plan whose `sellingPlanId` belongs to one of our groups → create DepositOrder + steps |
| `orders/updated`, `orders/edited` | Re-sync amounts |
| `order_transactions/create` | Re-sync, match transaction to step |
| `refunds/create` | Re-sync amounts |
| `fulfillments/create` | Set `fulfilledAt`; if balance trigger = ON_FULFILLMENT → enqueue `balance.request` (Module 4) |
| `orders/cancelled` | Status → CANCELLED, steps → CANCELLED, cancel reminders |

**Service `order-sync.server.ts` → `syncOrder(shop, orderGid)`**
1. Query order:
   ```graphql
   query Order($id: ID!) {
     order(id: $id) {
       id name cancelledAt displayFinancialStatus
       customer { id displayName email }
       currentTotalPriceSet { shopMoney { amount currencyCode } }
       totalReceivedSet     { shopMoney { amount } }
       totalRefundedSet     { shopMoney { amount } }
       totalOutstandingSet  { shopMoney { amount } }
       paymentTerms { paymentSchedules(first: 5) { nodes { id dueAt completedAt } } }
       paymentCollectionDetails { additionalPaymentCollectionUrl vaultedPaymentMethods { id } }
       lineItems(first: 100) { nodes { sellingPlan { sellingPlanId name } } }
       transactions(first: 50) { id kind status amountSet { shopMoney { amount } } processedAt gateway }
       tags
     }
   }
   ```
2. **Create** (first time): `ruleId` from `merchantCode`/`sellingPlanGroupGid`; steps:
   - Step 1 *Deposit*: amount = `totalReceived`, trigger AT_CHECKOUT, status PAID.
   - Step 2 *Balance*: amount = `totalOutstanding`, trigger = rule.balanceTrigger, status WAITING.
   - If trigger = DAYS_AFTER_ORDER → schedule `balance.request` at `createdAt + days`.
3. **Update**: overwrite `totalAmount/paidAmount/outstandingAmount` from Shopify (P1). Recompute step 2 amount = outstanding while step 2 not PAID. Match new successful transactions (kind `SALE`/`CAPTURE`, status `SUCCESS`) not yet in any `transactionGids` to the lowest unpaid step.
4. **Status rules** (in order): cancelled → CANCELLED · outstanding ≤ 0 → PAID (step 2 PAID, cancel reminders) · step FAILED → PAYMENT_FAILED · now > `balanceDueAt` and step REQUESTED → OVERDUE · step REQUESTED → BALANCE_REQUESTED · else DEPOSIT_PAID.
5. **Shopify writes** (only when changed):
   - Tags via `tagsAdd`/`tagsRemove`: `stagepay-deposit-paid`, `stagepay-balance-due`, `stagepay-overdue`, `stagepay-paid`.
   - App-owned metafield `$app:stagepay.schedule` (JSON: steps, statuses) for later admin extensions.
   - Order note: append a line per payment event via `orderUpdate(input:{id, note})` (e.g. `[Stagepay] 2026-11-03 Balance $7,000.00 paid by card`). The public API has no timeline-comment mutation, so the note is used instead of the timeline (adjusts FR-20).

**Daily job `orders.overdueScan`** (cron 06:00 shop time): REQUESTED steps with `balanceDueAt < now` → OVERDUE + staff alert (Module 5).

### 2.4 Frontend
None (Module 6 consumes).

### 2.5 Tests
| Type | Case |
|---|---|
| Unit | Status function: table-driven tests for every transition above |
| Unit | Transaction matching: deposit txn → step 1; later txn → step 2; refund doesn't mark paid |
| Integration | `orders/create` fixture with deposit plan → DepositOrder + 2 steps; without plan → ignored |
| Integration | Same webhook id twice → processed once |
| Integration | Order edited 20→25 tables → total $12,500, outstanding $9,500, step 2 amount $9,500 |
| Integration | Refund $1,000 → paidAmount updated, status unchanged unless outstanding = 0 |
| Integration | `orders/cancelled` → CANCELLED, scheduled jobs removed |
| Manual | Real test order on dev store → tags and note appear in Shopify admin |

### 2.6 Done criteria
- [ ] Every deposit order on the dev store appears in DB within 10 s with correct amounts.
- [ ] Edits/refunds/cancellations reflected; no double counting under duplicate webhooks.

---

## Module 3 — Storefront (theme block & validation)

### Purpose
Make the deposit option clear on the product page and enforce the minimum order value.

**Covers:** FR-7, FR-8, FR-3 (enforcement).

### 3.1 Design
Theme app block **"Stagepay deposit"** (product section), shown only when the product has one of our selling plans:

```
○ Pay in full – $1,000
● Pay 30% deposit now – $300 today, $700 before delivery
ⓘ Deposit is non-refundable once production starts.   [policy text]
```

- **The block renders its own deposit selector** (radio buttons "Pay in full" / "Pay X% deposit"). Feasibility test T10 showed Dawn exposes the selling plan in `product.selling_plan_groups` but renders **no picker**, so the selector is required, not a fallback.
- The selector writes the chosen plan into a hidden `<input name="selling_plan">` inside the theme's product form (created if missing; cleared for "Pay in full"), so the theme's own Add to cart sends `selling_plan` to `/cart/add`.
- The block shows the **today/later amounts** and **policy**, updating when the variant, quantity or choice changes.
- Block settings (theme editor): default choice (Pay in full / Deposit), show policy (on/off), text colour, "Hide selector if the theme already shows purchase options" (on/off, for themes with a native picker, to avoid two selectors).

### 3.2 Database
None. Policy text exposed through an **app-owned shop metafield** `$app:stagepay.policy` (written by Module 1/7 on save) so Liquid can read it as `app.metafields.stagepay.policy`.

### 3.3 Backend
- `metafields.server.ts`: `setShopAppMetafield(key, value)` via `metafieldsSet` on the app installation owner.
- **Cart & checkout validation Function** (`extensions/stagepay-min-order`): if any cart line has a deposit selling plan and cart subtotal < rule.minOrderValue → error "Deposit option requires an order of at least $5,000." Rule minimums are passed to the Function through a shop metafield `$app:stagepay.minimums` (`{sellingPlanId: amount}`), updated by `rules.sync`.

### 3.4 Frontend (extension files)
| File | Content |
|---|---|
| `extensions/stagepay-block/blocks/stagepay-deposit.liquid` | Reads `product.selling_plan_groups`, `selling_plan.checkout_charge.value_type` / `.value`, `product.selected_or_first_available_variant.price`; renders amounts with `money` filter |
| `extensions/stagepay-block/assets/stagepay.js` | Renders/handles the selector; sets the hidden `selling_plan` input in the product form; listens to variant/quantity changes (`change` on product form); recalculates amounts; no external requests |
| `extensions/stagepay-block/assets/stagepay.css` | Minimal styles inheriting theme fonts/colours |
| `extensions/stagepay-min-order/src/run.ts` | Validation Function |

### 3.5 Tests
| Type | Case |
|---|---|
| Unit | JS amount calculator: percentage, fixed per item, quantity changes |
| Unit | Validation Function: below min → error; above → none; no deposit plan → none |
| Manual | Dawn + 2 other OS 2.0 themes: block renders, amounts update with quantity, policy shows |
| Manual | Choose deposit → Add to cart → cart line shows the plan name; choose Pay in full → no plan on the line |
| Manual | Theme with a native picker + "Hide selector" on → only one selector visible |
| Manual | Lighthouse: product page performance score drops ≤ 2 points with block enabled |
| E2E (Playwright) | Buy product with deposit plan → checkout shows "Due today / Due later" amounts |

### 3.6 Done criteria
- [ ] Buyer sees correct today/later amounts and policy on supported themes.
- [ ] Checkout blocked below minimum order value with a clear message.

---

## Module 4 — Balance collection

### Purpose
Request and collect the remaining balance by saved card, payment link or recorded bank transfer, with automatic triggers and failure handling.

**Covers:** FR-13, FR-14, FR-16, FR-18, idempotency edge cases.

### 4.1 Design

**Order detail actions** (rendered in Module 6):
| Button | Shown when | Result |
|---|---|---|
| **Collect balance** (primary) | step 2 WAITING / FAILED / OVERDUE | Modal: "Collect $7,000.00 from Restaurant ABC" · method radio *Charge saved card •••• 4242* / *Email payment link* · **Confirm** |
| **Send payment link again** | step 2 REQUESTED with method PAYMENT_LINK | Resends email |
| **Record payment** | outstanding > 0 | Modal: amount (default = outstanding), method (Bank transfer / Cheque / Cash / Other), date, reference · **Record** |
| **Cancel balance** | outstanding > 0 | Confirm modal → step CANCELLED, reminders stop (order not changed in Shopify) |

Toasts: "Charge started", "Balance paid", "Charge failed — payment link sent to buyer".

### 4.2 Database

```prisma
model PaymentAttempt {
  id               String   @id @default(cuid())
  paymentStepId    String
  method           CollectionMethod
  amount           Decimal  @db.Decimal(12,2)
  idempotencyKey   String   @unique        // `${stepId}:${attemptNo}`
  status           AttemptStatus            // PENDING | SUCCEEDED | FAILED
  shopifyJobGid    String?
  paymentReferenceId String?
  errorCode        String?
  errorMessage     String?
  externalReference String?                // bank transfer ref
  createdByStaff   String?                 // staff user id / email
  createdAt        DateTime @default(now())
  completedAt      DateTime?
  step             PaymentStep @relation(fields: [paymentStepId], references: [id], onDelete: Cascade)
}

enum AttemptStatus { PENDING SUCCEEDED FAILED }
```

### 4.3 Backend

**Service `balance.server.ts`**

`requestBalance(depositOrderId, method, actor)` — job `balance.request`
1. Lock row (`SELECT … FOR UPDATE`); if step not in WAITING/FAILED/OVERDUE → no-op (prevents double click, FR edge case).
2. Re-sync order (Module 2) to get current outstanding; if 0 → mark PAID, stop.
3. `method = SAVED_CARD` and mandate available and feature enabled → `chargeSavedCard`; else → `sendPaymentLink`.
4. Step → REQUESTED, `requestedAt = now`, `balanceDueAt = now + order.overdueDays` (copied from the rule); schedule reminders (Module 5) unless reminders are off.

`chargeSavedCard(step)` — job `balance.charge`
```graphql
mutation Charge($id: ID!, $mandateId: ID!, $key: String!) {
  orderCreateMandatePayment(id: $id, mandateId: $mandateId, idempotencyKey: $key, autoCapture: true) {
    job { id done }
    paymentReferenceId
    userErrors { field message code }
  }
}
```
- Create `PaymentAttempt(PENDING)` **before** calling Shopify; `idempotencyKey` reused on retry of the same attempt.
- `amount` omitted in v1 (whole balance). (Partial amounts → Appendix C, depends on spike S4.)
- Poll job `balance.pollCharge` every 15 s up to 10 min with `orderPaymentStatus(orderId, paymentReferenceId) { status errorMessage }`:
  - `PURCHASED`/`AUTHORIZED`/`CAPTURED` → attempt SUCCEEDED → `syncOrder` (marks PAID).
  - `ERROR`/`VOIDED` → attempt FAILED → `handleFailure`.
  - Timeout → attempt stays PENDING, alert developer log; next `syncOrder` from webhook resolves it.

`sendPaymentLink(step)`
- URL = `order.paymentCollectionDetails.additionalPaymentCollectionUrl` (fallback: `orderInvoiceSend` if spike S5 failed).
- Store `paymentLinkUrl`, send email `BALANCE_DUE` (Module 5).

`recordManualPayment(depositOrderId, {amount, methodName, processedAt, reference}, actor)`
```graphql
mutation Manual($id: ID!, $amount: MoneyInput!, $name: String, $at: DateTime) {
  orderCreateManualPayment(id: $id, amount: $amount, paymentMethodName: $name, processedAt: $at) {
    order { id displayFinancialStatus totalOutstandingSet { shopMoney { amount } } }
    userErrors { field message }
  }
}
```
- Validate `0 < amount ≤ outstanding`.
- Attempt with method MANUAL_RECORD → SUCCEEDED → `syncOrder`. Partial manual amounts reduce outstanding; step stays REQUESTED until outstanding = 0.

`handleFailure(step, error)`
- Step FAILED, order PAYMENT_FAILED.
- Send buyer `CARD_FAILED` email with payment link; staff alert `STAFF_CARD_FAILED`.

**Triggers**
| Trigger | Source |
|---|---|
| MANUAL | Staff button |
| ON_FULFILLMENT | `fulfillments/create` webhook (Module 2) → `balance.request` with `order.collectionMethod` |
| DAYS_AFTER_ORDER (method) | Uses `order.collectionMethod` |
| DAYS_AFTER_ORDER | Scheduled job created at order creation (Module 2) |

**Routes**
| Route | Intent |
|---|---|
| `/app/orders/:id` POST | `collect` · `resend-link` · `record-payment` · `cancel-balance` |

### 4.4 Frontend
| File | Content |
|---|---|
| `components/CollectBalanceModal.tsx` | Method choice (card shows last 4 digits / brand if available), amount read-only, confirm |
| `components/RecordPaymentModal.tsx` | Amount, method select, date, reference; validation messages |
| `components/StepTimeline.tsx` | Steps with status badges and attempts history |

### 4.5 Tests
| Type | Case |
|---|---|
| Unit | Guard: request on PAID/CANCELLED/PROCESSING step → no-op |
| Unit | Manual amount validation (0, negative, > outstanding) |
| Integration | Card success: mutation mocked → poll returns PURCHASED → order PAID |
| Integration | Card failure: poll returns ERROR → FAILED, buyer + staff emails queued |
| Integration | Two concurrent `collect` requests → exactly one `orderCreateMandatePayment` call |
| Integration | Retry after network error uses same idempotency key |
| Integration | Manual $4,000 then $3,000 → PAID after second |
| Integration | ON_FULFILLMENT trigger fires once even if two fulfillments created |
| Manual (dev store) | Test card 4242 charge succeeds; decline card 4000 0000 0000 0002 fails; payment link paid in private window lands on same order |

### 4.6 Done criteria
- [ ] All three methods work on the non-Plus and Plus dev stores (or documented fallback from spike).
- [ ] No duplicate charges under double click, retries or duplicate webhooks.

---

## Module 5 — Emails & reminders

### Purpose
Send branded emails to buyers and staff, schedule reminders, log every email.

**Covers:** section 6 of requirements, FR-18 (emails part).

### 5.1 Design

**Email types**
| Key | To | Trigger | Default subject |
|---|---|---|---|
| `BALANCE_DUE` | Buyer | Balance requested by link | Your order {{order_name}} is ready – {{balance}} due |
| `BALANCE_CHARGED` | Buyer | Card charge succeeded | Payment received for order {{order_name}} |
| `REMINDER` | Buyer | Reminder days (default 3, 7, 14 after request) | Reminder: {{balance}} due for order {{order_name}} |
| `CARD_FAILED` | Buyer | Card charge failed | Action needed: payment for order {{order_name}} |
| `FULLY_PAID` | Buyer | Outstanding reaches 0 by link/manual | Order {{order_name}} is fully paid – thank you |
| `STAFF_CARD_FAILED` | Staff | Card charge failed | Card charge failed – {{order_name}} |
| `STAFF_OVERDUE` | Staff | Order becomes OVERDUE | Overdue balance – {{order_name}} ({{balance}}) |

**Placeholders:** `{{shop_name}} {{customer_name}} {{order_name}} {{order_total}} {{paid}} {{balance}} {{pay_link}} {{due_date}} {{policy}}`.

**Template editor** (`/app/settings/emails`): list of templates → edit subject + body (rich text limited to bold, links, line breaks) · "Insert placeholder" menu · live preview with sample order · **Send test** to staff email · **Reset to default**.

Layout: shop logo (from settings), body, button "Pay {{balance}}" (when `pay_link`), footer with shop name and reply-to.

### 5.2 Database

```prisma
model EmailTemplate {
  id        String   @id @default(cuid())
  shopId    String
  key       EmailKey
  subject   String
  body      String                     // stored as sanitized HTML subset
  enabled   Boolean  @default(true)
  updatedAt DateTime @updatedAt
  @@unique([shopId, key])
}

model EmailLog {
  id             String   @id @default(cuid())
  shopId         String
  depositOrderId String?
  key            EmailKey
  toEmail        String
  providerId     String?               // Postmark MessageID
  status         EmailStatus           // QUEUED | SENT | DELIVERED | BOUNCED | FAILED
  error          String?
  sentAt         DateTime?
  createdAt      DateTime @default(now())
  @@index([shopId, depositOrderId])
}

model Reminder {
  id            String   @id @default(cuid())
  paymentStepId String
  sendAt        DateTime
  sentAt        DateTime?
  cancelledAt   DateTime?
  jobId         String?
  @@index([sendAt])
}

enum EmailKey    { BALANCE_DUE BALANCE_CHARGED REMINDER CARD_FAILED FULLY_PAID STAFF_CARD_FAILED STAFF_OVERDUE }
enum EmailStatus { QUEUED SENT DELIVERED BOUNCED FAILED }
```

### 5.3 Backend
| Component | Responsibility |
|---|---|
| `email-provider.server.ts` | Interface `send({from, replyTo, to, subject, html, text, tag})`; Postmark implementation; test mode writes to log/Mailtrap |
| `email.server.ts` | `sendEmail(shopId, key, depositOrderId)`: load template (or default), render placeholders (escape HTML), insert EmailLog QUEUED → send → SENT/FAILED |
| `reminders.server.ts` | `scheduleReminders(step)` returns early if `settings.remindersEnabled = false`; otherwise creates Reminder rows for each `settings.reminderDays` (edited on P1) and pg-boss jobs `reminders.send` at `sendAt`; `cancelReminders(step)` sets `cancelledAt` and cancels jobs |
| Job `reminders.send` | Skip if step PAID/CANCELLED or reminder cancelled; else send `REMINDER` |
| Webhook `/webhooks/postmark` | Delivery/bounce events → update EmailLog status (verify basic-auth secret) |

From address: `"{{shop_name}}" <notifications@mail.<app-domain>>`; `Reply-To` = `settings.replyToEmail`.

### 5.4 Frontend
| File | Content |
|---|---|
| `app.settings.emails._index.tsx` | Template list with enabled toggles |
| `app.settings.emails.$key.tsx` | Editor, placeholder menu, preview iframe, send test, reset |

### 5.5 Tests
| Type | Case |
|---|---|
| Unit | Placeholder rendering, HTML escaping of customer name (`<script>` stays text) |
| Unit | Reminder schedule for days [3,7,14] from requestedAt |
| Integration | Step PAID → pending reminders cancelled, no email sent |
| Integration | Disabled template → not sent, logged as skipped |
| Integration | Provider error → EmailLog FAILED, job retried 3× |
| Manual | Emails in Mailtrap render correctly (Gmail, Outlook, mobile widths) |

### 5.6 Done criteria
- [ ] All 7 emails send with shop branding and correct amounts.
- [ ] Reminders stop immediately when paid or cancelled; every email visible in log.

---

## Module 6 — Dashboard & order detail

### Purpose
One place to see all deposit orders, money held and owed, and act on each order.

**Covers:** section 7 of requirements, FR-19 (warning).

### 6.1 Design

**Dashboard** — `/app` (home)
| Area | Content |
|---|---|
| Summary cards (4) | Deposits held (sum of paidAmount where status ≠ PAID/CANCELLED) · Balances owed (sum outstanding) · Overdue (sum outstanding where OVERDUE) · Paid this month (sum of step-2 payments in current month) |
| Tabs | All · Balance due · Requested · Overdue · Payment failed · Paid · Cancelled (with counts) |
| Table | Order · Customer · Total · Paid · Owed · Status badge · Balance due by · Last action |
| Search | Order name or customer name/email |
| **Reminders panel** | Card at the side/bottom: "Automatic reminders" switch (On/Off) · reminder days as chips (default 3, 7, 14; 1–60, ascending, max 5) · **Save**. Text: "Sent to buyers after a balance is requested, until it's paid." Turning off cancels all pending reminders; turning on schedules them for orders already in *Balance requested* |
| Paging | 50 per page, cursor based, sort by created (default) / owed / due date |
| Empty state | "No deposit orders yet" + link to rules + "Place a test order" guide |

**Order detail** — `/app/orders/:id`
| Area | Content |
|---|---|
| Header | Order name (link to Shopify order), customer, status badge |
| Warning banner | Critical banner when outstanding > 0: **"Do not ship: $7,000.00 unpaid"** |
| Money card | Total · Paid · Owed (from Shopify) · "Last synced 2 min ago" + Refresh |
| Steps | `StepTimeline` (Module 4) |
| Actions | Module 4 buttons |
| Emails | EmailLog list: type, to, status, date |
| Activity | AuditLog entries for this order (Module 9) |

Status badge colours: DEPOSIT_PAID info · BALANCE_REQUESTED attention · PAYMENT_FAILED critical · OVERDUE critical · PAID success · CANCELLED neutral.

### 6.2 Database
No new tables. Add indexes: `DepositOrder(shopId, createdAt)`, `DepositOrder(shopId, outstandingAmount)`, trigram index on `orderName`, `customerName`, `customerEmail` for search.

### 6.3 Backend
| Route | Loader |
|---|---|
| `/app` | `getSummary(shopId)` (one aggregate query) + `listOrders(shopId, {tab, q, cursor, sort})` + reminder settings; POST `intent=save-reminders` → validate, save to `ShopSettings`, cancel or schedule pending reminders |
| `/app/orders/:id` | `getOrderDetail(shopId, id)` (order + steps + attempts + emails + audit); POST `intent=refresh` → `syncOrder` |

Performance target: dashboard loader < 300 ms at 5,000 orders (indexes above; summary via single `SUM … FILTER (WHERE …)` query).

### 6.4 Frontend
| File | Content |
|---|---|
| `app._index.tsx` | Cards, tabs, IndexTable, search, pagination |
| `app.orders.$id.tsx` | Detail layout, banner, cards, modals from Module 4 |
| `components/StatusBadge.tsx` | Status → Polaris badge tone + label |
| `components/MoneyCard.tsx` | Total / paid / owed |

### 6.5 Tests
| Type | Case |
|---|---|
| Unit | Summary aggregation against seeded data (mixed statuses, currencies = shop currency) |
| Integration | Tab filters return correct rows; search by partial name; cursor paging stable |
| Integration | Shop A cannot load Shop B order id (404) |
| Integration | Reminders panel: turn off → pending reminders cancelled; turn on → scheduled for *Balance requested* orders; invalid days rejected |
| E2E | Seed 3 orders → dashboard counts → open detail → Collect balance → status updates |
| Perf | Seed 5,000 orders → loader < 300 ms (CI benchmark) |

### 6.6 Done criteria
- [ ] Dashboard totals match Shopify orders on the dev store.
- [ ] Every action reachable from order detail; warning visible on unpaid orders.

---

## Module 7 — Settings, onboarding & plan detection

### Purpose
Store-level configuration, first-run guide, and hiding features the store's Shopify plan cannot use.

### 7.1 Design

**Settings** — `/app/settings` (reminders now live on P1; collection method and days before overdue on each rule)
| Section | Fields |
|---|---|
| Branding | Logo upload (PNG/JPG ≤ 1 MB, stored via Shopify Files `stagedUploadsCreate` + `fileCreate`), brand colour |
| Emails | Reply-to email (default shop email), staff alert email(s) (comma list), link to template editor |
| Policy | Default deposit policy text (used when a rule has none) |

**Onboarding checklist** (dashboard top, dismissible, stored in settings):
1. Create your first deposit rule → `/app/rules/new`
2. Add the "Stagepay deposit" block to your product page → deep link `https://{shop}/admin/themes/current/editor?template=product&addAppBlockId={api_key}/stagepay-deposit&target=mainSection`
3. Place a test order → help text with test card numbers
4. Collect a test balance → link to the test order

**Plan notices:** if a feature is unavailable on the store's plan (from spike results, e.g. saved-card charge), show disabled option with tooltip "Not available on your Shopify plan — payment link is used instead".

### 7.2 Database

```prisma
model ShopSettings {
  id                 String   @id @default(cuid())
  shopId             String   @unique
  logoUrl            String?
  brandColor         String?  @default("#1F3864")
  replyToEmail       String
  staffAlertEmails   String[]
  remindersEnabled   Boolean  @default(true)        // edited on P1
  reminderDays       Int[]    @default([3, 7, 14])  // edited on P1
  defaultPolicyText  String?
  onboardingDone     Json     @default("{}")
  features           Json     @default("{}")   // { savedCardCharge: true, partialCharge: false }
  updatedAt          DateTime @updatedAt
}
```

### 7.3 Backend
- `settings.server.ts`: get-or-create defaults on install; zod validation (emails). Reminder validation (days 1–60 ascending, ≤ 5) is shared with the P1 Reminders panel.
- `plan.server.ts`: on install and on `shop/update` webhook → refresh `Shop.planName/isPlus`; compute `features` from plan + spike results table (constant in code).
- On save: write `$app:stagepay.policy` metafield (Module 3).

### 7.4 Frontend
| File | Content |
|---|---|
| `app.settings._index.tsx` | Form sections, save bar, logo uploader |
| `components/OnboardingChecklist.tsx` | 4 steps, progress, dismiss |
| `hooks/useFeature.ts` | `useFeature('savedCardCharge')` from loader data |

### 7.5 Tests
| Type | Case |
|---|---|
| Unit | Validation: invalid email; reminder days not ascending, > 5 reminders (P1 panel) |
| Integration | Install → default settings created with shop email |
| Integration | Plan change Grow → Plus via `shop/update` → features updated |
| Manual | Theme editor deep link adds the block |

### 7.6 Done criteria
- [ ] Settings persist and affect emails; P1 reminders panel and rule-level method/overdue days work.
- [ ] Unavailable features are hidden/explained per plan.

---

## Module 8 — App billing (Free / Pro)

### Purpose
Charge merchants through Shopify: **Free** (10 deposit orders per month) and **Pro** ($29/month, unlimited).

**Covers:** FR-21.

### 8.1 Design
**Plan page** — `/app/billing`: two plan cards (features, price, current plan badge), usage bar "7 of 10 deposit orders this month", **Upgrade** / **Downgrade** buttons.
Limit reached banner (dashboard): "You've used 10 of 10 free deposit orders this month. New deposit orders are still created in Shopify, but the app won't track them until you upgrade." → **Upgrade**.

### 8.2 Database

```prisma
model BillingState {
  id               String   @id @default(cuid())
  shopId           String   @unique
  plan             AppPlan  @default(FREE)     // FREE | PRO
  subscriptionGid  String?
  status           String?                     // ACTIVE, CANCELLED, ...
  periodStart      DateTime                    // usage window start (UTC month start)
  ordersThisPeriod Int      @default(0)
  updatedAt        DateTime @updatedAt
}
enum AppPlan { FREE PRO }
```

### 8.3 Backend
- Upgrade: `appSubscriptionCreate(name:"Pro", returnUrl, test: shop.isDevStore, lineItems:[{plan:{appRecurringPricingDetails:{price:{amount:29, currencyCode:USD}, interval: EVERY_30_DAYS}}}])` → redirect to `confirmationUrl`.
- Webhook `app_subscriptions/update` → update `plan/status`.
- Downgrade: `appSubscriptionCancel`.
- Usage: increment `ordersThisPeriod` when Module 2 creates a DepositOrder; reset monthly (job `billing.resetUsage`, 1st of month UTC).
- **Over limit on Free:** order is stored with flag `untracked = true` (add column), no reminders/actions; shown greyed with upgrade prompt. Deposits still work in Shopify (selling plans are not removed) so buyers are never blocked.
- **On upgrade to Pro:** `app_subscriptions/update` (ACTIVE) → set `untracked = false` on the current period's orders, re-sync each (Module 2) and schedule any due triggers/reminders.

### 8.4 Frontend
`app.billing.tsx`, `components/UsageBanner.tsx`.

### 8.5 Tests
| Type | Case |
|---|---|
| Unit | Usage window reset logic, limit check at exactly 10 / 11 |
| Integration | Upgrade flow with mocked `appSubscriptionCreate`; webhook ACTIVE → PRO |
| Integration | 11th order on Free → untracked, no reminder jobs |
| Integration | Upgrade to Pro → untracked orders become tracked and re-synced |
| Manual | Dev store test charge approve/decline |

### 8.6 Done criteria
- [ ] Free limit enforced; Pro unlocks unlimited; billing state follows Shopify webhooks.

---

## Module 9 — Compliance, security & audit log

### Purpose
Meet Shopify App Store requirements and keep a record of every money action.

### 9.1 Design
Activity list on order detail (Module 6): "Nov 3, 10:42 — Sarah (staff) collected balance by card — $7,000.00 — Succeeded".

### 9.2 Database

```prisma
model AuditLog {
  id             String   @id @default(cuid())
  shopId         String
  depositOrderId String?
  actorType      ActorType          // STAFF | SYSTEM | WEBHOOK
  actorId        String?            // staff user id / email
  action         String             // rule.created, balance.requested, payment.recorded, ...
  details        Json
  createdAt      DateTime @default(now())
  @@index([shopId, depositOrderId, createdAt])
}
enum ActorType { STAFF SYSTEM WEBHOOK }
```

### 9.3 Backend
| Item | Implementation |
|---|---|
| Audit | `audit(shopId, action, details, actor)` called from rules, balance, settings, billing services |
| GDPR `customers/data_request` | Collect DepositOrders/EmailLogs for the customer → email JSON export to shop owner email; log |
| GDPR `customers/redact` | Anonymise customer fields (name/email → `redacted`) in DepositOrder, EmailLog |
| GDPR `shop/redact` | Delete all shop rows (cascade) |
| `app/uninstalled` | Set `uninstalledAt`; schedule `shop.purge` in 48 h (cancelled on reinstall); stop all jobs for shop |
| Auth | Embedded app session tokens (template); every loader/action uses `authenticate.admin` and scopes queries by shop |
| Scopes | Minimal set (Appendix B) |
| Data | TLS everywhere; managed Postgres with encryption at rest and daily backups (7-day retention) |
| Secrets | Env vars only; no secrets in repo |
| Rate limiting | Per-shop limit on POST actions (e.g. 30/min) |

### 9.4 Frontend
`components/ActivityList.tsx`.

### 9.5 Tests
| Type | Case |
|---|---|
| Integration | Each GDPR webhook fixture → expected data change |
| Integration | Uninstall → purge after 48 h; reinstall within 48 h → purge cancelled |
| Integration | Collect balance creates audit entry with staff id |
| Security | Cross-shop access attempts on every route → 404 |
| Security | Webhook with bad HMAC → 401 |

### 9.6 Done criteria
- [ ] All mandatory webhooks implemented and tested; audit trail on every money action.

---

## Module 10 — End-to-end QA & launch

### 10.1 Acceptance tests (from merchant requirements; run on a Grow-plan and a Plus dev store)

| # | Test | Pass condition |
|---|---|---|
| A1 | Buyer pays 30% deposit on a $10,000 order | ONE order, $3,000 paid, $7,000 owed, *Partially paid* |
| A2 | Collect balance by saved card | Same order *Paid*, `BALANCE_CHARGED` email |
| A3 | Collect balance by link | Buyer pays on same order; no new order |
| A4 | Record $7,000 bank transfer | Order *Paid*, reminders stop |
| A5 | Balance trigger "on shipping" | Balance requested automatically when fulfilled |
| A6 | Reminders days 3/7/14 (time-compressed in test) | Sent, then stop on payment |
| A7 | Declined card | Buyer gets link email, staff alerted, status *Payment failed* |
| A8 | Double click Collect balance | Exactly one charge |
| A9 | Edit order 20 → 25 tables | Owed amount updates |
| A10 | Dashboard totals | Match Shopify orders |
| A11 | Free plan 11th order | Untracked + upgrade prompt; Pro billing works |
| A12 | Uninstall + GDPR webhooks | Data handled per Module 9 |

### 10.2 Non-functional checks
| Check | Target |
|---|---|
| Admin page load | < 2 s (5,000 orders) |
| Webhook response | < 1 s (work queued) |
| Product page impact | ≤ 2 Lighthouse points |
| Job failure alerting | Errors visible in error tracker within 5 min |

### 10.3 Launch checklist
- [ ] Production deploy, DB backups verified, error tracking and uptime alerts on.
- [ ] Production email domain verified (SPF, DKIM).
- [ ] App Store listing: name, tagline, description, 5 screenshots, demo video, support email, privacy policy URL.
- [ ] Setup guide + user guide published.
- [ ] Protected scopes approved by Shopify.
- [ ] Submit for review; beta merchants installed via direct link meanwhile.

---

## Appendix A — Full Prisma schema

Combined from Modules 0–9 (enums listed once).

```prisma
generator client { provider = "prisma-client-js" }
datasource db   { provider = "postgresql"; url = env("DATABASE_URL") }

model Shop {
  id String @id @default(cuid())
  shopDomain String @unique
  shopGid String @unique
  name String
  email String
  currencyCode String
  ianaTimezone String
  planName String
  isPlus Boolean @default(false)
  isDevStore Boolean @default(false)
  installedAt DateTime @default(now())
  uninstalledAt DateTime?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  settings ShopSettings?
  billing BillingState?
  rules DepositRule[]
  orders DepositOrder[]
}

model ShopSettings {
  id String @id @default(cuid())
  shopId String @unique
  shop Shop @relation(fields: [shopId], references: [id], onDelete: Cascade)
  logoUrl String?
  brandColor String? @default("#1F3864")
  replyToEmail String
  staffAlertEmails String[]
  remindersEnabled Boolean @default(true)
  reminderDays Int[] @default([3, 7, 14])
  defaultPolicyText String?
  onboardingDone Json @default("{}")
  features Json @default("{}")
  updatedAt DateTime @updatedAt
}

model BillingState {
  id String @id @default(cuid())
  shopId String @unique
  shop Shop @relation(fields: [shopId], references: [id], onDelete: Cascade)
  plan AppPlan @default(FREE)
  subscriptionGid String?
  status String?
  periodStart DateTime
  ordersThisPeriod Int @default(0)
  updatedAt DateTime @updatedAt
}

model DepositRule {
  id String @id @default(cuid())
  shopId String
  shop Shop @relation(fields: [shopId], references: [id], onDelete: Cascade)
  name String
  depositType DepositType
  depositValue Decimal @db.Decimal(12, 2)
  minOrderValue Decimal? @db.Decimal(12, 2)
  balanceTrigger StepTrigger
  balanceTriggerDays Int?
  finalDueDays Int @default(90)
  collectionMethod CollectionMethod @default(SAVED_CARD)
  overdueDays Int @default(7)
  buyerLabel String
  policyText String?
  active Boolean @default(true)
  sellingPlanGroupGid String?
  sellingPlanGid String?
  syncStatus SyncStatus @default(PENDING)
  syncError String?
  deletedAt DateTime?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  targets RuleTarget[]
  @@unique([shopId, name])
  @@index([shopId, active])
}

model RuleTarget {
  id String @id @default(cuid())
  ruleId String
  rule DepositRule @relation(fields: [ruleId], references: [id], onDelete: Cascade)
  targetType TargetType
  targetGid String
  title String
  @@unique([ruleId, targetGid])
}

model DepositOrder {
  id String @id @default(cuid())
  shopId String
  shop Shop @relation(fields: [shopId], references: [id], onDelete: Cascade)
  orderGid String
  orderName String
  ruleId String?
  customerGid String?
  customerName String?
  customerEmail String?
  currencyCode String
  totalAmount Decimal @db.Decimal(12, 2)
  paidAmount Decimal @db.Decimal(12, 2)
  outstandingAmount Decimal @db.Decimal(12, 2)
  status DepositOrderStatus
  untracked Boolean @default(false)
  balanceDueAt DateTime?
  finalDueAt DateTime
  mandateGid String?
  collectionMethod CollectionMethod
  overdueDays Int
  paymentLinkUrl String?
  fulfilledAt DateTime?
  cancelledAt DateTime?
  lastSyncedAt DateTime
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  steps PaymentStep[]
  emails EmailLog[]
  @@unique([shopId, orderGid])
  @@index([shopId, status])
  @@index([shopId, balanceDueAt])
  @@index([shopId, createdAt])
  @@index([shopId, outstandingAmount])
}

model PaymentStep {
  id String @id @default(cuid())
  depositOrderId String
  depositOrder DepositOrder @relation(fields: [depositOrderId], references: [id], onDelete: Cascade)
  position Int
  label String
  amount Decimal @db.Decimal(12, 2)
  trigger StepTrigger
  triggerDays Int?
  status PaymentStepStatus
  method CollectionMethod?
  requestedAt DateTime?
  paidAt DateTime?
  transactionGids String[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  attempts PaymentAttempt[]
  reminders Reminder[]
  @@unique([depositOrderId, position])
}

model PaymentAttempt {
  id String @id @default(cuid())
  paymentStepId String
  step PaymentStep @relation(fields: [paymentStepId], references: [id], onDelete: Cascade)
  method CollectionMethod
  amount Decimal @db.Decimal(12, 2)
  idempotencyKey String @unique
  status AttemptStatus
  shopifyJobGid String?
  paymentReferenceId String?
  errorCode String?
  errorMessage String?
  externalReference String?
  createdByStaff String?
  createdAt DateTime @default(now())
  completedAt DateTime?
}

model Reminder {
  id String @id @default(cuid())
  paymentStepId String
  step PaymentStep @relation(fields: [paymentStepId], references: [id], onDelete: Cascade)
  sendAt DateTime
  sentAt DateTime?
  cancelledAt DateTime?
  jobId String?
  @@index([sendAt])
}

model EmailTemplate {
  id String @id @default(cuid())
  shopId String
  key EmailKey
  subject String
  body String
  enabled Boolean @default(true)
  updatedAt DateTime @updatedAt
  @@unique([shopId, key])
}

model EmailLog {
  id String @id @default(cuid())
  shopId String
  depositOrderId String?
  depositOrder DepositOrder? @relation(fields: [depositOrderId], references: [id], onDelete: Cascade)
  key EmailKey
  toEmail String
  providerId String?
  status EmailStatus
  error String?
  sentAt DateTime?
  createdAt DateTime @default(now())
  @@index([shopId, depositOrderId])
}

model AuditLog {
  id String @id @default(cuid())
  shopId String
  depositOrderId String?
  actorType ActorType
  actorId String?
  action String
  details Json
  createdAt DateTime @default(now())
  @@index([shopId, depositOrderId, createdAt])
}

model WebhookEvent {
  id String @id @default(cuid())
  shopId String
  webhookId String @unique
  topic String
  receivedAt DateTime @default(now())
  processedAt DateTime?
  error String?
  @@index([shopId, topic])
}

enum DepositType { PERCENTAGE FIXED }
enum TargetType { PRODUCT COLLECTION }
enum SyncStatus { PENDING SYNCED FAILED }
enum DepositOrderStatus { DEPOSIT_PAID BALANCE_REQUESTED PAYMENT_FAILED OVERDUE PAID CANCELLED }
enum PaymentStepStatus { WAITING REQUESTED PROCESSING PAID FAILED CANCELLED }
enum StepTrigger { AT_CHECKOUT MANUAL ON_FULFILLMENT DAYS_AFTER_ORDER }
enum CollectionMethod { SAVED_CARD PAYMENT_LINK MANUAL_RECORD }
enum AttemptStatus { PENDING SUCCEEDED FAILED }
enum EmailKey { BALANCE_DUE BALANCE_CHARGED REMINDER CARD_FAILED FULLY_PAID STAFF_CARD_FAILED STAFF_OVERDUE }
enum EmailStatus { QUEUED SENT DELIVERED BOUNCED FAILED }
enum AppPlan { FREE PRO }
enum ActorType { STAFF SYSTEM WEBHOOK }
```

---

## Appendix B — Shopify scopes, webhooks & APIs

⚠ Confirm exact scope names and field availability against the pinned API version during the Module 0 spike.

**Access scopes**
| Scope | Used for |
|---|---|
| `read_products` | Resource picker, collection expansion |
| `read_purchase_options`, `write_purchase_options` | Selling plan groups |
| `read_orders`, `write_orders` | Order sync, tags, note, manual payments |
| `read_payment_terms` | `order.paymentTerms` (balance due date) — required, found in feasibility test T2 |
| `read_customers` | Buyer name/email on orders |
| `read_payment_mandate`, `write_payment_mandate` | Saved-card balance charge (may need Shopify approval) |
| `read_fulfillments` | Fulfillment trigger |
| `write_files` | Logo upload |

**Webhooks**
`orders/create` · `orders/updated` · `orders/edited` · `orders/cancelled` · `order_transactions/create` · `refunds/create` · `fulfillments/create` · `shop/update` · `app/uninstalled` · `app_subscriptions/update` · `customers/data_request` · `customers/redact` · `shop/redact`

**Admin GraphQL operations**
| Operation | Module |
|---|---|
| `shop { plan { displayName partnerDevelopment shopifyPlus } … }` | 0, 7 |
| `sellingPlanGroupCreate / Update / Delete / AddProducts / RemoveProducts` | 1 |
| `collection.products` | 1 |
| `order(id)` (fields in Module 2) | 2 |
| `tagsAdd`, `tagsRemove`, `orderUpdate` (note), `metafieldsSet` | 2, 3, 7 |
| `orderCreateMandatePayment`, `orderPaymentStatus` | 4 |
| `orderCreateManualPayment` | 4 |
| `paymentCollectionDetails.additionalPaymentCollectionUrl`, `orderInvoiceSend` (fallback) | 4 |
| `stagedUploadsCreate`, `fileCreate` | 7 |
| `appSubscriptionCreate`, `appSubscriptionCancel` | 8 |

**Extensions**
| Extension | Type | Module |
|---|---|---|
| `stagepay-block` | Theme app extension (app block) | 3 |
| `stagepay-min-order` | Cart & checkout validation Function | 3 |

---

## Appendix C — Later phases (not in v1)

Designed-for but not built in v1. The schema already supports N `PaymentStep`s.

| Feature | Notes / dependency |
|---|---|
| 3+ payment steps (e.g. 40/30/30) | Middle steps need partial charge (spike S4) or manual record; payment link for partial amounts per spike S5 |
| Deposits on quotes / draft orders | Draft order deposit fields (Plus, API 2026-07); non-Plus via checkout link carrying selling plan — to verify |
| Deposit then net 30 | Payment terms after fulfillment |
| Admin order-page block | Admin UI extension reading `$app:stagepay.schedule` metafield |
| Customer account extension | Schedule + "Pay balance" in new customer accounts |
| PDF invoices & statements | Per payment step + open-balance statement |
| QuickBooks / Xero sync | Deposits as customer-deposit liability |
| Shopify Flow triggers & actions | deposit paid, balance due, overdue, fully paid / request balance, charge balance |
| Block overdue customers | Validation Function using customer metafield |
| Plus in-checkout extension | Schedule + policy checkbox in checkout |
| SMS reminders | Twilio provider behind the email-provider style interface |
| Order-level fixed deposit | Fixed amount per order instead of per item |
