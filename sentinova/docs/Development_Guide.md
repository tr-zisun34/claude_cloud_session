# Sentinova – Spam Protection

**Development Guide** · v1.0 · Companion to the Product Specification v0.1

---

## 0. How to read this document

This guide is the **build contract**. The Product Specification says *what* the merchant sees. This guide says *how* to build it, at a level where a developer doesn't need to make architectural decisions while coding.

- Sections 1–3 cover the decisions and the overall shape of the system.
- Sections 4–15 are the implementation spec: data model, rule engine, the two evaluation paths, actions, UI, jobs and APIs.
- Sections 16–21 cover quality, operations, the plan and the verification spikes.

Conventions:

- **MUST / SHOULD / MAY** are used in the RFC 2119 sense.
- 🔎 **Verify** marks a Shopify API detail that has to be confirmed against the pinned API version during the M0 spikes (section 21). The design always includes a fallback, so a failed verification does not block the architecture.
- Every code block is a **reference implementation**: the intended shape and behaviour. It is not meant to be pasted as-is.

---

## 1. Architecture decisions (changes from guide v0.1)

These decisions resolve the gaps and contradictions between the spec and guide v0.1. Items marked **[needs product sign-off]** change behaviour that the spec describes.

### 1.1 Data model: 4 app tables (not 7)

| Table (spec / v0.1) | Decision | Why |
| --- | --- | --- |
| `Session` | **Keep** (template-owned) | Required by `@shopify/shopify-app-session-storage-prisma`. |
| `Shop` | **Keep**, extended | Holds settings, plan, function activation IDs, snapshot sync state and the usage counter. Adding these as columns avoids four small side tables. |
| `Rule` | **Keep** | Conditions and actions are stored as validated JSON on the rule. |
| `RuleCondition` | **Drop** | Conditions are always read and written together with their rule, never queried on their own, and have to be serialised to JSON for the Function snapshot anyway. A child table would add joins, ordering bugs and transactional writes for no benefit. Integrity comes from the zod schema (section 5.5). |
| `MatchLog` | **Keep, restructured**: **one row per order (or checkout event)**, not one per rule | The Orders page, Approve/Reject, money-saved accounting and idempotency all work per order. One row per rule double-counted money and made "rule(s) triggered" a group-by. Matched rules are stored as `ruleIds[]` plus a JSON snapshot. |
| `ChargebackRecord` | **Keep** (renamed `Chargeback`) | Needed for time-window counts (30/90/365 days), for manual imports when the store isn't on Shopify Payments, and for redaction. |
| `DailyStats` | **Drop** | At the stated scale (≤ 10k log rows per shop page set) dashboard aggregates are millisecond-level SQL over an indexed `MatchLog`. A rollup table also goes stale whenever a row changes later (reject → saved, undo → unsaved, an estimate finalised after 24 h), which would need a recompute pipeline. Add a materialised view only if p95 dashboard latency goes above 300 ms. |
| `ListEntry` | **Drop for v1** | List values (domains, IPs, prefixes) live inside the condition that uses them. Reusable named lists are a v1.2 feature ("Block/allow lists") and get their own table then. The global disposable-domain list is reference data, not tenant data, so it lives in Redis plus a bundled file (section 10.2). |

The rule engine also **derives** address mismatch from the two raw addresses and chargeback counts from `Chargeback` rows, so neither needs to be stored.

### 1.2 Behavioural decisions

| # | Decision | Rationale |
| --- | --- | --- |
| D1 | **Drop the `orders/updated` webhook.** Late-arriving risk data is handled with a **deferred evaluation** (re-queue with delay) instead. | `orders/updated` fires on every edit, fulfilment and payment event, which is roughly 5–20× the volume of `orders/create`. It also makes it hard to guarantee "never action twice". |
| D2 | **Auto restock is an option of Cancel (and Reject), not a standalone action.** **[needs product sign-off]** | Inventory on an open order is *committed*. Adding stock back while the order stays open creates phantom stock and overselling. In the UI, "Auto restock" stays as a checkbox under Cancel & refund. This also removes the `write_inventory` scope. |
| D3 | **Evaluate all matching rules first, then execute one merged action plan.** | The spec says both "execute on match" (step 3) and "most restrictive wins" (step 5). These only agree if the plan is merged before anything runs. |
| D4 | **Three-valued (Kleene) logic for conditions.** Unknown data never makes a negated operator true. | `is_not_disposable` with an unknown email must not match. Kleene logic also lets an "inconclusive" result be logged distinctly ("Skipped – data unavailable"). |
| D5 | **"Unknown counts as match" applies to After-checkout rules only.** Before checkout, unknown is always no-match. | Fail-open at checkout is a hard requirement (spec §5). A guest who hasn't typed an email yet must never lose COD. |
| D6 | **New installs start in dry-run (log-only).** | This builds trust and shows a "would have saved" figure. The onboarding flow nudges the merchant to go live. |
| D7 | **Approve / Reject ships in the MVP**, not v1.1. **[needs product sign-off]** | "In review" is an MVP action. Without Approve/Reject a held order can only be released from the Shopify admin, and Sentinova would never learn the outcome. |
| D8 | **No PII stored in clear.** Hashes are used for matching, masked values for display, and customer names are fetched live from Shopify for the visible page. | Smaller compliance surface, simpler redaction, and less to leak. |
| D9 | **One `$app` shop metafield snapshot** feeds both Functions. Sync is inline on save, with a retry job and an hourly reconciliation. | One write per change, one source of truth, and a visible error state for the merchant. |
| D10 | **Default: stop on first match** (spec open question 5). | Predictable for merchants. Most-restrictive-wins applies only when a merchant turns "Stop processing" off. |
| D11 | **Disable Pay now = blocking validation error** (spec open question 3). | It works regardless of which payment methods exist. |
| D12 | **POS orders are excluded by default** (setting). | Orders taken in person aren't spam, and POS orders don't get a risk analysis. |

### 1.3 Answers to spec open questions (proposed defaults)

| Spec Q | Proposed answer |
| --- | --- |
| 1. Plans at launch | All plans for After-checkout. Before-checkout actions depend on what the M0 spike S1 finds. Any action a plan can't use is greyed out with an explanation. |
| 2. Phone/IP providers | Pick in M6 after comparing pricing (section 10). Lookups are counted per shop so usage add-ons stay possible. |
| 3. Disable Pay now | Validation error with a message (D11). |
| 4. A rule spanning both stages | No. A rule belongs to one stage. |
| 5. Multiple matches | Stop on first match by default (D10). |
| 6. Retention | 12 months for `MatchLog`, configurable from 3 to 24. `Chargeback` rows are kept for 25 months, which covers the 365-day window plus a buffer. |
| 7. Shared blocklist | Out of scope for v1. |

---

## 2. Prerequisites

- A Shopify Partner account and **three development stores**:
  1. `sentinova-dev-payments`: Shopify Payments in test mode (Bogus Gateway) to test refunds, disputes and Protect.
  2. `sentinova-dev-cod`: Cash on Delivery enabled plus a manual payment method in a second language to test name matching.
  3. `sentinova-dev-plus` (Plus sandbox / development store with Plus features, if available): to test checkout UI extensions on the Information, Shipping and Payment steps.
- Node.js 22 LTS, pnpm 9+, Git, Docker (for local Postgres and Redis).
- Shopify CLI (latest): `npm i -g @shopify/cli`.
- A Sentry project and a host with a stable HTTPS URL that runs **two process types** (web and worker), for example Fly.io, Render or Railway.
- Before M6: accounts with a phone line-type provider and an IP intelligence provider.

---

## 3. System architecture

```
                         ┌──────────────────────────── Shopify ────────────────────────────┐
 Merchant (embedded)     │  Admin GraphQL API      Webhooks          Checkout                │
        │                │        ▲                   │          ┌───────────────────────┐ │
        ▼                │        │                   │          │ Functions (Wasm)      │ │
 ┌──────────────┐        │        │                   │          │  cod-payment-custom.  │◄┼── reads $app:sentinova.
 │  WEB process │────────┼────────┘                   │          │  checkout-validation  │ │   checkout_rules (shop mf)
 │  React Router│  rules/settings/approve/reject      │          └───────────────────────┘ │   chargebacks (customer mf)
 │  + Polaris   │────────┼─ metafieldsSet (snapshot) ─┼──────────────────────▲              │
 │  /webhooks/* │◄───────┼────────────────────────────┘                      │              │
 └──────┬───────┘        └───────────────────────────────────────────────────┼──────────────┘
        │ enqueue (BullMQ)                                                    │
        ▼                                                                     │
 ┌──────────────┐   facts    ┌──────────────────┐                             │
 │ Redis        │◄──────────►│ WORKER process   │── snapshot retry/reconcile ─┘
 │ queue+cache  │            │ evaluate→execute │── Admin GraphQL (tag/hold/cancel/metafields)
 └──────────────┘            │ maintenance jobs │── Enrichment adapters (email/phone/IP)
        ▲                    └────────┬─────────┘
        │                             ▼
        └──────────────────── PostgreSQL (Prisma): Session, Shop, Rule, MatchLog, Chargeback
```

### 3.1 Two evaluation paths, one engine

| | Before checkout | After checkout |
| --- | --- | --- |
| Runs in | Shopify Function (Wasm sandbox, no network, no clock) | Worker process |
| Trigger | Buyer interacts with checkout | `orders/create` webhook |
| Rules source | Compact snapshot in a shop metafield | Postgres |
| Facts source | Function input (cart, buyer identity, customer metafield) and the snapshot (disposable-domain filter) | Order GraphQL query, `Chargeback` table, enrichment adapters |
| Unknown handling | Always no-match (D5) | Shop setting `unknownAsMatch` |
| Failure mode | **Fail open**: return no operations | Retry with backoff, then `FAILED` and an alert |
| Logging | None from the Function itself (section 9.6) | `MatchLog` row |

The evaluator is `packages/rule-engine`: **pure TypeScript with zero runtime dependencies and no Node or DOM APIs**, so it compiles into the Function bundle unchanged. zod schemas live in a separate entry point (`@sentinova/rule-engine/schema`) so zod never gets into the Function bundle.

### 3.2 Process model

| Process | Command | Scales by | Responsibilities |
| --- | --- | --- | --- |
| `web` | `pnpm start` | HTTP load | Admin UI, auth, webhook intake (verify → enqueue → 200), Approve/Reject/Undo, inline snapshot sync |
| `worker` | `pnpm worker` | Queue depth | Order evaluation and actions, dispute processing, snapshot retries, scheduled maintenance |

Webhook handlers MUST NOT call Shopify or providers. They verify the HMAC, enqueue, and return `200` within 1 s.

---

## 4. Repository layout

```
sentinova/
├─ shopify.app.toml
├─ shopify.app.staging.toml
├─ package.json                 # pnpm workspace root (the app itself lives at root, as in the template)
├─ pnpm-workspace.yaml          # packages/*, extensions/*
├─ .env.example
├─ app/
│  ├─ shopify.server.ts         # shopifyApp() config, afterAuth hook → services/install
│  ├─ db.server.ts              # Prisma client singleton
│  ├─ routes/
│  │  ├─ app.tsx                       # layout + nav (Dashboard, Rules, Orders, Settings)
│  │  ├─ app._index.tsx                # Dashboard
│  │  ├─ app.onboarding.tsx
│  │  ├─ app.rules._index.tsx          # list, toggle, reorder, duplicate, delete
│  │  ├─ app.rules.new.tsx
│  │  ├─ app.rules.$id.tsx
│  │  ├─ app.orders._index.tsx         # table + detail drawer
│  │  ├─ app.orders.$id.tsx            # approve / reject / undo actions (POST only)
│  │  ├─ app.orders.export.tsx         # CSV
│  │  ├─ app.settings.tsx
│  │  ├─ app.billing.tsx
│  │  ├─ webhooks.orders.tsx
│  │  ├─ webhooks.disputes.tsx
│  │  ├─ webhooks.fulfillment-orders.tsx
│  │  ├─ webhooks.app.tsx              # app/uninstalled, app_subscriptions/update, shop/update
│  │  ├─ webhooks.compliance.tsx
│  │  ├─ api.checkout-events.tsx       # v1.1, called by checkout UI extension
│  │  └─ healthz.tsx
│  ├─ services/
│  │  ├─ install.server.ts       # afterAuth: upsert shop, activate functions, metafield defs, backfill
│  │  ├─ shop.server.ts          # settings read/write (zod), plan info
│  │  ├─ rules.server.ts         # CRUD, priority, validation, triggers snapshot sync
│  │  ├─ snapshot.server.ts      # compile + metafieldsSet + hash
│  │  ├─ facts.server.ts         # order → Facts
│  │  ├─ evaluate.server.ts      # load rules, run engine, build plan, write MatchLog
│  │  ├─ executor.server.ts      # tag / hold / release / cancel with per-action idempotency
│  │  ├─ decisions.server.ts     # approve / reject / undo
│  │  ├─ chargebacks.server.ts   # dispute upsert, counts, customer metafield
│  │  ├─ analytics.server.ts     # dashboard SQL
│  │  ├─ billing.server.ts       # plan limits + usage
│  │  ├─ privacy.server.ts       # hashing, masking, redaction
│  │  └─ enrichment/
│  │     ├─ types.ts             # EmailChecker / PhoneChecker / IpChecker
│  │     ├─ email.server.ts
│  │     ├─ phone.server.ts
│  │     ├─ ip.server.ts
│  │     └─ cache.server.ts      # Redis cache + circuit breaker
│  ├─ graphql/                   # .graphql documents + codegen output (typed)
│  ├─ jobs/
│  │  ├─ queues.server.ts        # queue definitions, job names, default options
│  │  ├─ worker.ts               # worker entrypoint (pnpm worker)
│  │  └─ handlers/*.ts           # one file per job name (section 14)
│  └─ lib/                       # logger, errors, money, ids, time
├─ packages/
│  └─ rule-engine/
│     ├─ src/index.ts            # evaluate, types, registry (zero deps)
│     ├─ src/registry.ts         # FILTERS / ACTIONS metadata (single source of truth)
│     ├─ src/operators.ts
│     ├─ src/plan.ts             # action merge
│     ├─ src/snapshot.ts         # encode/decode compact snapshot
│     ├─ src/bloom.ts            # bloom filter (FNV-1a), base64
│     ├─ src/net.ts              # IPv4/IPv6 + CIDR
│     ├─ src/normalize.ts        # email/phone/address normalisation
│     ├─ src/schema.ts           # zod schemas (separate entry: @sentinova/rule-engine/schema)
│     └─ test/                   # unit tests + shared golden fixtures
├─ extensions/
│  ├─ cod-payment-customization/  # Function (JS)
│  ├─ checkout-validation/        # Function (JS), v1.1
│  └─ checkout-reporter/          # Checkout UI extension, v1.1, Plus only
├─ prisma/
│  ├─ schema.prisma
│  └─ migrations/
├─ data/disposable-domains/       # bundled fallback lists (full + checkout subset)
├─ scripts/                       # build-bloom.ts, replay-webhooks.ts, seed-dev.ts
└─ tests/
   ├─ integration/
   └─ e2e/                        # Playwright
```

---

## 5. Project setup

```bash
shopify app init --name sentinova-spam-protection          # React Router template, TypeScript
cd sentinova-spam-protection

shopify app generate extension --template payment_customization   --name cod-payment-customization --flavor vanilla-js
shopify app generate extension --template cart_checkout_validation --name checkout-validation      --flavor vanilla-js

pnpm add bullmq ioredis zod pino @sentry/node libphonenumber-js
pnpm add -D vitest @vitest/coverage-v8 tsx playwright @graphql-codegen/cli @shopify/api-codegen-preset

docker compose up -d        # postgres:16, redis:7
pnpm prisma migrate dev
shopify app dev             # web; in a second terminal: pnpm worker:dev
```

Functions are written in **JavaScript (TypeScript compiled with Javy)** so they can share the engine package. The trade-off is instruction count: the M0 spike S3 measures it, and a performance budget is set in section 9.5.

### 5.1 `shopify.app.toml`

```toml
name = "Sentinova - Spam Protection"
client_id = "<from partner dashboard>"
application_url = "https://app.sentinova.example"
embedded = true

[access_scopes]
# write_* implies read_*. Each scope is justified in section 15.2.
scopes = "write_orders,write_customers,write_merchant_managed_fulfillment_orders,write_third_party_fulfillment_orders,read_shopify_payments_disputes,write_payment_customizations,write_validations"

[webhooks]
api_version = "2026-10"   # pin the latest stable at project start, upgrade quarterly

[[webhooks.subscriptions]]
topics = ["orders/create", "orders/cancelled"]
uri = "/webhooks/orders"
# POS exclusion is done in the worker (settings.excludedSources), not with a webhook filter,
# so merchants can change it without an app deploy.

[[webhooks.subscriptions]]
topics = ["disputes/create", "disputes/update"]
uri = "/webhooks/disputes"

[[webhooks.subscriptions]]
topics = ["fulfillment_orders/hold_released"]
uri = "/webhooks/fulfillment-orders"

[[webhooks.subscriptions]]
topics = ["app/uninstalled", "app_subscriptions/update", "shop/update"]
uri = "/webhooks/app"

[[webhooks.subscriptions]]
compliance_topics = ["customers/data_request", "customers/redact", "shop/redact"]
uri = "/webhooks/compliance"

[auth]
redirect_urls = ["https://app.sentinova.example/auth/callback"]

[pos]
embedded = false
```

### 5.2 Environment variables

| Variable | Used by | Notes |
| --- | --- | --- |
| `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET` | web, worker | From the Partner Dashboard. |
| `SHOPIFY_APP_URL` | web | Public HTTPS URL. |
| `SCOPES` | web | Must equal the toml scopes. |
| `DATABASE_URL` | web, worker | Postgres, `?connection_limit=10`. |
| `REDIS_URL` | web, worker | `maxmemory-policy noeviction` (BullMQ requirement). |
| `HASH_PEPPER` | web, worker | 32+ random bytes, base64. **Never rotate without a rehash plan**, because hashes would stop matching. |
| `SENTRY_DSN`, `LOG_LEVEL` | web, worker | |
| `PHONE_PROVIDER_KEY`, `IP_PROVIDER_KEY` | worker | v1.1. |
| `DISPOSABLE_LIST_URL` | worker | Upstream list source for the daily refresh. |

---

## 6. Database (Prisma)

### 6.1 Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ── Template-owned. Do not modify the shape. ─────────────────────────────
model Session {
  id            String    @id
  shop          String
  state         String
  isOnline      Boolean   @default(false)
  scope         String?
  expires       DateTime?
  accessToken   String
  userId        BigInt?
  firstName     String?
  lastName      String?
  email         String?
  accountOwner  Boolean   @default(false)
  locale        String?
  collaborator  Boolean?  @default(false)
  emailVerified Boolean?  @default(false)
  @@index([shop])
}

// ── Tenant ───────────────────────────────────────────────────────────────
model Shop {
  id                     String      @id @default(cuid())
  domain                 String      @unique                 // xyz.myshopify.com
  shopGid                String      @unique                 // gid://shopify/Shop/123
  name                   String
  currency               String      @db.Char(3)             // shop (not presentment) currency
  timezone               String                              // IANA, used for dashboard day buckets
  shopifyPlan            String?                             // plan.publicDisplayName
  isPlus                 Boolean     @default(false)
  isDevStore             Boolean     @default(false)
  billingPlan            BillingPlan @default(FREE)
  settings               Json        @default("{}")          // ShopSettings, zod-validated (6.3)

  paymentCustomizationId String?                             // gid, our COD customization
  validationId           String?                             // gid, our checkout validation (v1.1)

  snapshotHash           String?                             // sha256 of last successfully written snapshot
  snapshotSyncedAt       DateTime?
  snapshotError          String?                             // shown as a banner when non-null

  usagePeriodStart       DateTime    @default(now())
  ordersEvaluated        Int         @default(0)             // in current usage period (billing.server.ts)

  onboardedAt            DateTime?
  installedAt            DateTime    @default(now())
  uninstalledAt          DateTime?

  rules                  Rule[]
  matchLogs              MatchLog[]
  chargebacks            Chargeback[]
}

model Rule {
  id            String    @id @default(cuid())
  shopId        String
  shop          Shop      @relation(fields: [shopId], references: [id], onDelete: Cascade)
  name          String    @db.VarChar(80)
  stage         Stage
  matchType     MatchType @default(ALL)
  priority      Int                                          // 1..n, dense, per (shopId, stage)
  enabled       Boolean   @default(true)
  stopOnMatch   Boolean   @default(true)
  conditions    Json                                         // Condition[] (1..10), see 7.2
  actions       Json                                         // Action[] (1..4),   see 7.3
  schemaVersion Int       @default(1)                        // bump when Condition/Action JSON shape changes
  version       Int       @default(1)                        // optimistic locking for concurrent edits
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@index([shopId, stage, priority])
}

// One row per evaluated order (stage AFTER) or per checkout event (stage BEFORE).
// Rows are written only when at least one rule matched OR was inconclusive.
model MatchLog {
  id            String     @id @default(cuid())
  shopId        String
  shop          Shop       @relation(fields: [shopId], references: [id], onDelete: Cascade)
  stage         Stage
  orderId       String?                                      // gid; null for checkout events
  orderName     String?                                      // "#1001"
  checkoutRef   String?                                      // hashed checkout token (checkout events)
  customerId    String?                                      // gid, used for redaction only
  emailHash     String?                                      // HMAC-SHA256(pepper, normalised email)
  phoneHash     String?
  maskedEmail   String?                                      // "j•••@gmail.com"

  status        LogStatus
  dryRun        Boolean    @default(false)
  ruleIds       String[]                                     // matched rule ids (may reference deleted rules)
  evaluation    Json                                         // EvaluationSnapshot (6.4), immutable after write
  actions       Json       @default("[]")                    // ActionState[] (6.4), mutated by executor
  timeline      Json       @default("[]")                    // TimelineEvent[] (6.4), append-only

  orderTotal    Decimal?   @db.Decimal(14, 2)                // shop currency at evaluation time
  currency      String     @db.Char(3)
  amountSaved   Decimal    @default(0) @db.Decimal(14, 2)
  savedKind     SavedKind  @default(NONE)

  decidedBy     String?                                      // staff user id (approve/reject/undo)
  decidedAt     DateTime?
  error         String?                                      // last human-readable error
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @updatedAt

  @@unique([shopId, stage, orderId])                         // idempotency: one AFTER row per order
  @@index([shopId, createdAt(sort: Desc), id])               // Orders page cursor + dashboard ranges
  @@index([shopId, status])
  @@index([shopId, emailHash])
  @@index([ruleIds], type: Gin)
}

model Chargeback {
  id          String           @id @default(cuid())
  shopId      String
  shop        Shop             @relation(fields: [shopId], references: [id], onDelete: Cascade)
  source      ChargebackSource
  disputeId   String?                                        // Shopify dispute gid (null for MANUAL)
  orderId     String?
  customerId  String?
  emailHash   String?
  phoneHash   String?
  amount      Decimal          @db.Decimal(14, 2)
  currency    String           @db.Char(3)
  status      String                                         // Shopify dispute status, or "manual"
  initiatedAt DateTime
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt

  @@unique([shopId, disputeId])
  @@index([shopId, customerId])
  @@index([shopId, emailHash])
  @@index([shopId, phoneHash])
}

enum Stage            { BEFORE_CHECKOUT AFTER_CHECKOUT }
enum MatchType        { ALL ANY }
enum BillingPlan      { FREE GROWTH PRO }
enum ChargebackSource { SHOPIFY MANUAL }
enum SavedKind        { NONE ACTUAL ESTIMATED PENDING_ESTIMATE WOULD_HAVE }
enum LogStatus {
  PROCESSING        // row written, actions being executed
  TAGGED
  IN_REVIEW
  APPROVED          // in-review order released by staff (in app or in Shopify admin)
  CANCELLED         // auto-cancelled, rejected, or cancelled externally while in review
  REVERTED          // undo applied
  SKIPPED           // actions not applicable, or inconclusive (unknown data)
  FAILED            // ≥1 action failed after retries → manual action needed
  DRY_RUN
  CHECKOUT_BLOCKED  // v1.1, from checkout reporter
  COD_HIDDEN        // v1.1, from checkout reporter
}
```

### 6.2 Notes on the schema

- **Tenant isolation:** every query in `services/*` MUST filter by `shopId`, and the `shopId` MUST come from the authenticated session, never from the request body. A Prisma extension (`withShop(shopId)`) adds the filter automatically. A unit test asserts that every repository function needs it.
- **Priority:** there is no unique constraint on `(shopId, stage, priority)`, because Prisma can't defer it. Priority changes run in a transaction that first takes a row lock on the shop (`SELECT … FROM "Shop" WHERE id=$1 FOR UPDATE`) and then renumbers 1..n. Lists are small (≤ 50), so a full renumber is fine.
- **Nullable unique:** Postgres treats NULLs as distinct, so many checkout-event rows (`orderId = NULL`) can coexist with the unique index.
- **Deleted rules:** rules are hard-deleted. `MatchLog.evaluation` keeps the rule name and conditions as they were, so history stays readable.
- **Money:** `Decimal(14,2)` in shop currency. Inside the engine, amounts are compared as integer **hundredths** (`Math.round(x*100)`) on both sides, which is exact for 0- and 2-decimal currencies and consistent for 3-decimal ones.

### 6.3 `ShopSettings` (JSON in `Shop.settings`)

```ts
export const ShopSettings = z.object({
  dryRun:            z.boolean().default(true),                         // D6
  unknownAsMatch:    z.boolean().default(false),                        // AFTER stage only (D5)
  codMethodNames:    z.array(z.string().trim().min(1).max(100)).max(20)
                      .default(['Cash on Delivery (COD)', 'Cash on Delivery']),
  blockMessage:      z.string().trim().min(1).max(250)
                      .default("We couldn't process this order. Please contact support."),
  reviewTag:         TagSchema.default('sentinova-in-review'),
  approvedTag:       TagSchema.default('sentinova-approved'),
  excludedSources:   z.array(z.enum(['pos', 'draft_order', 'api'])).default(['pos']),   // D12
  retentionMonths:   z.number().int().min(3).max(24).default(12),
  notifyOnFailure:   z.boolean().default(true),                         // email shop owner on FAILED
});
export const TagSchema = z.string().trim().min(1).max(40).regex(/^[^,]+$/);
```

Read settings with `ShopSettings.parse(shop.settings)`, so defaults fill in any keys that are missing. Never read raw JSON.

### 6.4 JSON column shapes

```ts
// MatchLog.evaluation — what was decided and why (immutable)
interface EvaluationSnapshot {
  engineVersion: string;                 // package version, for debugging
  unknownAsMatch: boolean;
  rules: Array<{
    id: string; name: string; priority: number; matchType: 'ALL' | 'ANY';
    result: 'MATCH' | 'INCONCLUSIVE';
    conditions: Array<{ filter: string; operator: string; expected: unknown;
                        actual: string | number | boolean | null;   // masked/derived, never raw PII
                        result: 'true' | 'false' | 'unknown' }>;
  }>;
  plan: ActionPlan;                       // merged plan (7.6)
  deferred?: number;                      // how many times evaluation was deferred for risk data
}

// MatchLog.actions — one entry per planned action, mutated by executor
interface ActionState {
  type: 'tag' | 'hold' | 'cancel';
  state: 'PENDING' | 'DONE' | 'SKIPPED' | 'FAILED' | 'REVERTED';
  attempts: number;
  params: Record<string, unknown>;        // e.g. { tags: [...] } | { reason, refund, restock, notify }
  ref?: { holdIds?: string[]; jobId?: string; tagsAdded?: string[] };   // needed for undo/release
  reason?: string;                        // SKIPPED/FAILED human-readable reason
  at?: string;                            // ISO time of last state change
}

// MatchLog.timeline — append-only human history for the detail drawer
interface TimelineEvent {
  at: string;
  kind: 'EVALUATED' | 'DEFERRED' | 'ACTION_DONE' | 'ACTION_SKIPPED' | 'ACTION_FAILED'
      | 'APPROVED' | 'REJECTED' | 'UNDONE' | 'CANCELLED_EXTERNALLY' | 'HOLD_RELEASED_EXTERNALLY';
  actor: 'system' | { staffId: string; name?: string };
  detail?: string;
}
```

Appends use a single SQL statement so concurrent writers can't lose events:
`UPDATE "MatchLog" SET timeline = timeline || $1::jsonb, "updatedAt" = now() WHERE id = $2`.

---

## 7. Rule engine (`packages/rule-engine`)

### 7.1 Filter registry: single source of truth

The UI form, zod validation, snapshot compiler and docs all read the same registry. Adding a filter means one registry entry, one operator implementation and tests.

| Filter key | Label | Stages | Release | Facts used |
| --- | --- | --- | --- | --- |
| `order_total` | Order total | BEFORE, AFTER | MVP | `orderTotal` |
| `email` | Email | BEFORE, AFTER | MVP | `email` |
| `chargeback_count` | Customer chargebacks | BEFORE\*, AFTER | MVP | `chargebacks` |
| `risk_level` | Shopify risk level | AFTER | MVP | `riskLevel` |
| `address_mismatch` | Billing / shipping mismatch | AFTER | MVP | `billing`, `shipping` |
| `shopify_protect` | Shopify Protect eligibility | AFTER | MVP | `protect` |
| `phone` | Phone number | BEFORE\*\*, AFTER | v1.1 | `phone` |
| `ip` | IP address | AFTER | v1.1 | `ip` |

\* Before checkout only for **logged-in customers** (it reads the customer metafield). For guests it's unknown, so no match. The rule builder shows this as a hint.
\*\* Before checkout only `starts_with`, `equals` and `contains` are offered. Line-type status needs a provider.

**Final stage matrix (replaces spec §3.2):** address mismatch is After-only, because the billing address isn't in the Function input at the payment step. IP is After-only in v1.

### 7.2 Condition operators and value schemas

| Filter | Operator | Value (JSON) | True when | Unknown when |
| --- | --- | --- | --- | --- |
| `order_total` | `eq` `gt` `gte` `lt` `lte` | `{ "amount": "300.00" }` | comparison on hundredths | total missing |
| | `between` | `{ "min": "100.00", "max": "500.00" }` | `min ≤ total ≤ max` (inclusive) | total missing |
| `email` | `is_disposable` / `is_not_disposable` | `{}` | domain is (not) disposable | no email, or disposable lookup unavailable |
| | `domain_equals` | `{ "domain": "mail.ru" }` | exact match after normalisation | no email |
| | `domain_contains` | `{ "text": "temp" }` | substring | no email |
| | `domain_in_list` | `{ "domains": ["a.com", "b.net"] }` (≤ 500) | in set | no email |
| `chargeback_count` | `eq` `gt` `gte` `lt` | `{ "count": 1, "window": "90d" }`, window ∈ `30d,90d,365d,all` | count in window vs `count` | no customer identity at all |
| `risk_level` | `is` / `is_not` | `{ "level": "HIGH" }` | equal / not equal | risk `NONE`, `PENDING` or missing |
| | `is_any_of` | `{ "levels": ["MEDIUM","HIGH"] }` | in set | same |
| `address_mismatch` | `is_mismatched` | `{ "fields": ["country","province","city","zip","name"] }` (≥ 1) | **any** selected field differs | Kleene over fields (see below) |
| | `is_matched` | same | **all** selected fields equal | same |
| `shopify_protect` | `is` / `is_not` | `{ "status": "ELIGIBLE" }` (`ELIGIBLE`, `NOT_ELIGIBLE`, `PENDING`) | equal / not equal | Protect data missing |
| `phone` | `status_is` / `status_is_not` | `{ "statuses": ["INVALID","VOIP"] }` | in set / not in set | no phone or lookup unavailable |
| | `starts_with` | `{ "prefix": "+23480" }` | E.164 prefix | no phone |
| | `equals` | `{ "number": "+2348012345678" }` | E.164 equal | no phone |
| | `contains` | `{ "digits": "0000" }` | digit substring | no phone |
| `ip` | `equals` | `{ "ip": "203.0.113.7" }` | equal (v4/v6 canonical) | no IP |
| | `in_range` | `{ "ranges": ["203.0.113.0/24","2001:db8::/32","198.51.100.9"] }` (≤ 500) | inside any range (bare IP = /32 or /128) | no IP |
| | `country_is` / `country_is_not` | `{ "countries": ["NG","GH"] }` ISO-3166 α-2 | in / not in set | no IP or lookup unavailable |
| | `is_proxy` / `is_not_proxy` | `{}` | provider flags proxy, VPN, Tor or hosting | no IP or lookup unavailable |

**Normalisation (in `normalize.ts`, shared by both paths):**

- Email: trim, lowercase, take the domain after the **last** `@`, strip a trailing `.`. Gmail dots and `+tags` are **not** stripped for matching (that's a v1.2 option).
- Phone: E.164 via `libphonenumber-js` on the server, with the shipping country as the default region. In the Function, only strip spaces, dashes and brackets, and require a leading `+`.
- Address fields: Unicode NFKD, strip diacritics, lowercase, collapse whitespace. `zip`: uppercase and drop spaces and dashes. `country` and `province` compare **codes**. `name` = `first + " " + last`.
- Address-mismatch Kleene logic: for each selected field, both values present → equal or not; either missing → unknown. `is_mismatched` = any `true` → true, else any unknown → unknown, else false. `is_matched` = the negation of that, with unknown kept as unknown.
- The **risk level** is taken from Shopify's own assessment (`provider == null`): the highest of `LOW < MEDIUM < HIGH`. `NONE` is unknown. `PENDING` triggers deferral (section 9.2).

### 7.3 Actions

| Action key | Stage | Params | Notes |
| --- | --- | --- | --- |
| `hide_cod` | BEFORE | `{ note?: string }` | Hides every payment method whose name matches `settings.codMethodNames` (case-insensitive, exact after trim). |
| `block_checkout` | BEFORE | `{ message?: string(≤250), note?: string }` | v1.1. Falls back to `settings.blockMessage`. |
| `tag` | AFTER | `{ tags: string[1..5] }` | Default `["sentinova-flagged"]`. |
| `in_review` | AFTER | `{}` | Adds `settings.reviewTag` and holds all open fulfillment orders. |
| `cancel` | AFTER | `{ reason: 'FRAUD'\|'DECLINED'\|'OTHER', notifyCustomer: boolean = true, refund: boolean = true, restock: boolean = true, staffNote?: string(≤200) }` | "Auto restock" is the `restock` flag (D2). |

`note` is an internal label shown in the Orders page. It's never shown to the buyer.

### 7.4 Types

```ts
export type Stage = 'BEFORE_CHECKOUT' | 'AFTER_CHECKOUT';
export type Tri = true | false | 'unknown';

export interface Condition { filter: FilterKey; operator: string; value: Record<string, unknown>; }
export type Action =
  | { type: 'hide_cod'; params: { note?: string } }
  | { type: 'block_checkout'; params: { message?: string; note?: string } }
  | { type: 'tag'; params: { tags: string[] } }
  | { type: 'in_review'; params: Record<string, never> }
  | { type: 'cancel'; params: { reason: 'FRAUD' | 'DECLINED' | 'OTHER'; notifyCustomer: boolean;
                                refund: boolean; restock: boolean; staffNote?: string } };

export interface RuleDef {
  id: string; name: string; stage: Stage; priority: number;
  matchType: 'ALL' | 'ANY'; stopOnMatch: boolean;
  conditions: Condition[]; actions: Action[];
}

export interface Address { country?: string; province?: string; city?: string; zip?: string; name?: string; }

/** Collected signals. `undefined` = unknown. */
export interface Facts {
  orderTotal?: number;                                         // shop currency, major units
  email?: { domain: string; disposable?: boolean };
  phone?: { e164: string; status?: 'VALID' | 'INVALID' | 'VOIP' };
  ip?: { address: string; country?: string; proxy?: boolean };
  chargebacks?: { d30: number; d90: number; d365: number; all: number };
  riskLevel?: 'LOW' | 'MEDIUM' | 'HIGH';
  billing?: Address;
  shipping?: Address;
  protect?: 'ELIGIBLE' | 'NOT_ELIGIBLE' | 'PENDING';
}
```

### 7.5 Evaluator

```ts
// Kleene combinators
const and = (xs: Tri[]): Tri => xs.includes(false) ? false : xs.includes('unknown') ? 'unknown' : true;
const or  = (xs: Tri[]): Tri => xs.includes(true)  ? true  : xs.includes('unknown') ? 'unknown' : false;
const not = (x: Tri): Tri => x === 'unknown' ? 'unknown' : !x;

export function evalCondition(c: Condition, f: Facts): Tri {
  const op = OPERATORS[c.filter]?.[c.operator];
  if (!op) return 'unknown';                 // unknown filter/operator (e.g. newer snapshot) → never match
  return op(c.value, f);
}

// operators.ts (excerpt): every operator returns Tri and handles missing facts itself
export const OPERATORS: Record<FilterKey, Record<string, (v: any, f: Facts) => Tri>> = {
  order_total: {
    gt:  (v, f) => cmp(f.orderTotal, v.amount, (a, b) => a > b),
    gte: (v, f) => cmp(f.orderTotal, v.amount, (a, b) => a >= b),
    lt:  (v, f) => cmp(f.orderTotal, v.amount, (a, b) => a < b),
    lte: (v, f) => cmp(f.orderTotal, v.amount, (a, b) => a <= b),
    eq:  (v, f) => cmp(f.orderTotal, v.amount, (a, b) => a === b),
    between: (v, f) => f.orderTotal === undefined ? 'unknown'
      : hundredths(f.orderTotal) >= hundredths(v.min) && hundredths(f.orderTotal) <= hundredths(v.max),
  },
  email: {
    is_disposable:     (_, f) => f.email?.disposable ?? 'unknown',
    is_not_disposable: (_, f) => not(f.email?.disposable ?? 'unknown'),
    domain_equals:     (v, f) => f.email ? f.email.domain === normDomain(v.domain) : 'unknown',
    domain_contains:   (v, f) => f.email ? f.email.domain.includes(String(v.text).toLowerCase()) : 'unknown',
    domain_in_list:    (v, f) => f.email ? (v.domains as string[]).map(normDomain).includes(f.email.domain) : 'unknown',
  },
  chargeback_count: {
    gt:  (v, f) => cbCmp(f, v, (n, k) => n > k),
    gte: (v, f) => cbCmp(f, v, (n, k) => n >= k),
    lt:  (v, f) => cbCmp(f, v, (n, k) => n < k),
    eq:  (v, f) => cbCmp(f, v, (n, k) => n === k),
  },
  // risk_level, address_mismatch, shopify_protect, phone, ip: same pattern, see 7.2
};

const hundredths = (x: number | string) => Math.round(Number(x) * 100);
const cmp = (fact: number | undefined, val: string, fn: (a: number, b: number) => boolean): Tri =>
  fact === undefined ? 'unknown' : fn(hundredths(fact), hundredths(val));
const WINDOW = { '30d': 'd30', '90d': 'd90', '365d': 'd365', all: 'all' } as const;
const cbCmp = (f: Facts, v: any, fn: (n: number, k: number) => boolean): Tri =>
  f.chargebacks === undefined ? 'unknown' : fn(f.chargebacks[WINDOW[v.window as keyof typeof WINDOW]], Number(v.count));

export function evalRule(rule: RuleDef, f: Facts) {
  const results = rule.conditions.map(c => evalCondition(c, f));
  const outcome = rule.matchType === 'ALL' ? and(results) : or(results);
  return { outcome, results };
}

export interface Hit { rule: RuleDef; outcome: Tri; results: Tri[]; }

/**
 * Returns matched rules (outcome resolved with unknownAsMatch) plus inconclusive ones (for logging).
 * Rules MUST already be filtered to one stage and to enabled=true.
 */
export function evaluate(rules: RuleDef[], f: Facts, opts: { unknownAsMatch: boolean }) {
  const matched: Hit[] = [];
  const inconclusive: Hit[] = [];
  for (const rule of [...rules].sort((a, b) => a.priority - b.priority)) {
    const { outcome, results } = evalRule(rule, f);
    const isMatch = outcome === true || (outcome === 'unknown' && opts.unknownAsMatch);
    if (!isMatch) { if (outcome === 'unknown') inconclusive.push({ rule, outcome, results }); continue; }
    matched.push({ rule, outcome, results });
    if (rule.stopOnMatch) break;
  }
  return { matched, inconclusive };
}
```

### 7.6 Action plan (merge)

```ts
export interface ActionPlan {
  // AFTER stage
  tags: string[];                                     // union of all matched tag actions (deduped, ≤ 10)
  inReview: boolean;
  cancel?: { reason: 'FRAUD'|'DECLINED'|'OTHER'; notifyCustomer: boolean; refund: boolean; restock: boolean; staffNote?: string };
  // BEFORE stage
  hideCod: boolean;
  block?: { message: string };
}

export function buildPlan(hits: Hit[], settings: { blockMessage: string; reviewTag: string }): ActionPlan
```

Merge rules (deterministic):

1. `cancel` beats `in_review`. If any matched rule cancels, `inReview = false`, and the review tag isn't added.
2. `cancel` params come from the **highest-priority** matched rule that has `cancel`.
3. Tags are unioned in priority order. The review tag is added only when `inReview` is true.
4. `block` beats `hideCod` (if checkout is blocked, hiding COD is irrelevant). Each Function still evaluates on its own (9.3 and 9.4).
5. The `block` message comes from the highest-priority rule with `block_checkout`, falling back to `settings.blockMessage`.

### 7.7 Validation (zod, `schema.ts`)

Validation runs on rule save, on snapshot compile and at worker load. Rules that fail validation at load are skipped and reported to Sentry, never thrown. Checks:

- `conditions` has 1–10 items; `actions` has 1–4; `name` is 1–80 characters.
- Each filter is allowed for the rule's stage; each operator is allowed for the filter; the value matches the operator's schema (7.2): amounts are positive decimals with ≤ 2 dp, `min ≤ max`, IPs and CIDRs are valid, country codes are ISO-α2, domains are valid hostnames, lists hold ≤ 500 entries with no duplicates.
- Actions are allowed for the stage. There's no `in_review` + `cancel` in the same rule (hard error. The spec says "warn", but a rule can't do both, so the UI shows the conflict inline and blocks save). There are no duplicate action types.
- Filters marked `release: 'v1.1'` are rejected until the feature flag `features.v11` is on for the shop.
- **Plan limits** (section 13) are checked in `rules.server.ts`, not in the schema.

### 7.8 Parity guarantee

`packages/rule-engine/test/golden/*.json` holds `{ rules, facts, opts, expected }` cases. The **same files** run in the engine unit tests, in the Function tests (`shopify app function run` with fixture inputs built from them) and in the worker integration tests. CI fails if any path disagrees.

---

## 8. Install, onboarding and uninstall

### 8.1 `afterAuth` (services/install.server.ts)

Runs on every OAuth completion. Every step MUST be **idempotent**, because reinstalls and re-auth happen.

1. `query { shop { id name myshopifyDomain currencyCode ianaTimezone plan { publicDisplayName shopifyPlus partnerDevelopment } } }` → upsert `Shop` and clear `uninstalledAt`.
2. **Metafield definitions** (create if missing):
   - Shop: `$app:sentinova` / `checkout_rules`, type `json`, admin access `MERCHANT_READ`. 🔎 Verify that Function input can read it (spike S2).
   - Customer: `$app:sentinova` / `chargebacks`, type `json`, admin access `MERCHANT_READ`.
3. **Activate the COD Function:** list `paymentCustomizations` and look for one with our `functionId` / handle. If none exists, `paymentCustomizationCreate({ title: "Sentinova – COD protection", enabled: true, functionHandle: "cod-payment-customization" })`. 🔎 Verify `functionHandle` against `functionId`; the fallback is to look up the id with `shopifyFunctions(apiType: "payment_customization")`. Store the gid in `Shop.paymentCustomizationId`. If creation fails because of the plan, store the error and show it in onboarding.
4. (v1.1) Do the same for `validationCreate({ functionHandle: "checkout-validation", enable: true, blockOnFailure: false })`. **`blockOnFailure: false` is what makes the validation fail open.**
5. Write an empty snapshot (`{v:1, dry:true, r:[]}`) so the Functions always find valid JSON.
6. Enqueue `disputes.backfill` (jobId `backfill:{shopId}`).
7. Redirect to `/app/onboarding` when `onboardedAt` is null.

### 8.2 Onboarding (`/app/onboarding`)

The flow has four steps, and the merchant can skip any of them:

1. **Welcome:** explains dry-run, which is on (D6).
2. **Checkout protection status:** shows whether the COD customization is active, which plan the store is on, and which Before-checkout actions are available (from step 3 of 8.1 and spike S1).
3. **COD method names:** an editable list pre-filled with the defaults, with the help text "Type the payment method name exactly as buyers see it at checkout". 🔎 Verify whether any Admin API lists manual payment method names (spike S6). If one does, pre-fill from it.
4. **Starter rule:** pick one of three templates and create it:
   - "Hide COD for high-value orders from disposable emails" (BEFORE: `order_total gt 200` ALL `email is_disposable` → `hide_cod`).
   - "Review high-risk orders" (AFTER: `risk_level is HIGH` → `tag` + `in_review`).
   - "Cancel repeat chargeback customers" (AFTER: `chargeback_count gte 2 window all` → `cancel` FRAUD, refund, restock).

Finishing the flow sets `onboardedAt`.

### 8.3 Uninstall and reinstall

- `app/uninstalled`: set `uninstalledAt`, delete `Session` rows for the shop, and remove the shop's queued jobs (`queue.getJobs(['delayed','waiting'])` filtered by shop). Workers check `uninstalledAt` and no-op.
- Shopify deletes the app's `$app` metafields and Function customizations on uninstall. 🔎 Verify (spike S7). If it doesn't, nothing can be done after uninstall anyway, so instead make step 3 of 8.1 tolerate leftovers on reinstall.
- Data is deleted 48 h later by `shop/redact` (section 15.4). Reinstalling within 48 h restores everything.

---

## 9. Before-checkout path

### 9.1 Snapshot format (`snapshot.ts`)

The Functions read one compact JSON document. Keys are short because of input size limits.

```jsonc
{
  "v": 1,                                  // format version; Function ignores unknown versions → fail open
  "dry": false,                            // shop dryRun → Functions return no operations
  "cod": ["cash on delivery (cod)", "cash on delivery"],   // normalised codMethodNames
  "msg": "We couldn't process this order. Please contact support.",
  "r": [                                   // enabled BEFORE rules, pre-sorted by priority
    { "i": "ckx1", "m": "A", "s": 1,       // id, matchType A|Y, stopOnMatch 1|0
      "c": [["order_total","gt",{"amount":"200.00"}], ["email","is_disposable",{}]],
      "a": [["hide_cod"]] }
  ],
  "d": { "m": 65536, "k": 10, "b": "<base64 bits>" },      // disposable-domain bloom filter
  "fm": ["gmail.com","outlook.com","yahoo.com","icloud.com","hotmail.com"]  // freemail allowlist, checked before bloom
}
```

- **Budget:** the snapshot MUST be ≤ **12 KB** serialised, and the compiler rejects saves that exceed it ("Too many before-checkout conditions or list entries"). 🔎 Verify the real metafield and Function input limits in spike S2, then adjust the budget to 50% of the limit.
- **Disposable bloom filter:** built from the **checkout subset** list (the ~5,000 most common disposable domains, `data/disposable-domains/checkout.txt`) with a target false-positive rate of 0.1%, which is about 9 KB. Hashing is FNV-1a 32-bit with Kirsch–Mitzenmacher double hashing, implemented once in `bloom.ts`. Freemail domains are checked first and are **never** disposable, which removes the most damaging false positives.
- `snapshotHash = sha256(json)`. If the hash hasn't changed, the sync skips the write.

### 9.2 Snapshot sync (`snapshot.server.ts`)

**Triggers:** rule create, update, delete, toggle or reorder in the BEFORE stage; settings changes to `dryRun`, `codMethodNames` or `blockMessage`; the daily disposable-list refresh; and the hourly reconcile.

```ts
export async function syncSnapshot(admin: AdminApi, shop: Shop): Promise<void> {
  const rules = await loadEnabledRules(shop.id, 'BEFORE_CHECKOUT');
  const settings = ShopSettings.parse(shop.settings);
  const json = encodeSnapshot(rules, settings, await disposableBloom());   // throws SnapshotTooLarge
  const hash = sha256(json);
  if (hash === shop.snapshotHash) return;

  const res = await admin.graphql(METAFIELDS_SET, { variables: { m: [{
    ownerId: shop.shopGid, namespace: '$app:sentinova', key: 'checkout_rules', type: 'json', value: json,
  }] } });
  const errs = res.data.metafieldsSet.userErrors;
  if (errs.length) throw new SyncError(errs.map(e => e.message).join('; '));
  await db.shop.update({ where: { id: shop.id },
    data: { snapshotHash: hash, snapshotSyncedAt: new Date(), snapshotError: null } });
}
```

- In route actions, the DB commit happens first and `syncSnapshot` is called after it. On failure: set `snapshotError`, enqueue `snapshot.sync` (jobId `snapshot:{shopId}`, so retries collapse), and return the error as a toast. The rule is saved; the banner reads "Checkout rules not yet active – retrying".
- `snapshot.reconcile` runs hourly over active shops whose `snapshotSyncedAt` is older than their latest BEFORE rule `updatedAt` or that have a `snapshotError`.

### 9.3 COD payment customization Function

`extensions/cod-payment-customization/src/run.graphql`

```graphql
query RunInput {
  shop { metafield(namespace: "$app:sentinova", key: "checkout_rules") { value } }
  presentmentCurrencyRate                       # 🔎 Verify field availability (spike S2)
  cart {
    cost { totalAmount { amount currencyCode } }
    buyerIdentity {
      email
      phone
      customer { metafield(namespace: "$app:sentinova", key: "chargebacks") { value } }
    }
  }
  paymentMethods { id name }
}
```

`extensions/cod-payment-customization/src/run.ts`

```ts
import { decodeSnapshot, evaluate, buildPlan, bloomHas, factsFromCart } from '@sentinova/rule-engine';

const NO_CHANGES = { operations: [] };

export function run(input: RunInput) {
  try {
    const raw = input.shop?.metafield?.value;
    if (!raw) return NO_CHANGES;
    const snap = decodeSnapshot(raw);                       // returns null on unknown v / bad JSON
    if (!snap || snap.dry) return NO_CHANGES;

    const rules = snap.rules.filter(r => r.actions.some(a => a.type === 'hide_cod'));
    if (!rules.length) return NO_CHANGES;

    const codIds = input.paymentMethods
      .filter(m => snap.cod.includes(m.name.trim().toLowerCase()))
      .map(m => m.id);
    if (!codIds.length) return NO_CHANGES;                  // nothing to hide: skip evaluation (saves instructions)

    const facts = factsFromCart(input, snap);               // shared helper, see below
    const { matched } = evaluate(rules, facts, { unknownAsMatch: false });   // D5
    if (!matched.length) return NO_CHANGES;

    return { operations: codIds.map(id => ({ hide: { paymentMethodId: id } })) };  // 🔎 Verify output shape for pinned version
  } catch {
    return NO_CHANGES;                                      // FAIL OPEN
  }
}
```

`factsFromCart` (in the engine package):

- `orderTotal = Number(cart.cost.totalAmount.amount) / (presentmentCurrencyRate ?? 1)`. **Cart amounts are in the buyer's presentment currency, while rule amounts are in shop currency.** If the rate field is unavailable and the currency differs from the shop currency, `orderTotal` is set to `undefined` (unknown), never compared across currencies. The shop currency is included in the snapshot (`"cur"`) for this check.
- `email`: domain from `buyerIdentity.email`. `disposable` = not in `fm` AND `bloomHas(d, domain)` OR in any `domain_in_list` value. Undefined if no email has been entered yet.
- `chargebacks`: parsed from the customer metafield (`{d30,d90,d365,all,asOf}`). Undefined if there's no customer. A known customer without a metafield gets `{0,0,0,0}`.
- `phone`: digits of `buyerIdentity.phone` (v1.1).

### 9.4 Checkout validation Function: "Disable Pay now" (v1.1)

- Same input plus `buyerJourney { step }` and `localization { language { isoCode } }`.
- Evaluate rules that have `block_checkout`. On a match, return `{ operations: [{ validationAdd: { errors: [{ message, target: "$.cart" }] } }] }`. 🔎 Verify the output shape.
- Only block when `buyerJourney.step` is `CHECKOUT_INTERACTION` or `CHECKOUT_COMPLETION` (not `CART_INTERACTION`, so the cart page is unaffected).
- Activated with `blockOnFailure: false` (8.1), so a Function error never blocks a checkout.

### 9.5 Function performance and size budget

| Metric | Budget | Enforced by |
| --- | --- | --- |
| Instructions per run (20 rules × 10 conditions, 500-entry lists) | ≤ 50% of Shopify's limit | CI step: `shopify app function run` on the worst-case fixture, parse the reported instruction count |
| Wasm size | ≤ 50% of Shopify's limit | CI step: check the build output size |
| Snapshot size | ≤ 12 KB (adjust after S2) | `encodeSnapshot` throws |

Engine code rules for the Function path: no regex compiled per run, no `JSON.parse` of nested strings, no zod, no `Intl`.

### 9.6 Before-checkout analytics

Functions can't make network calls or write logs anywhere the app can read. Strategy:

| Release | Mechanism | What the merchant sees |
| --- | --- | --- |
| MVP | None. The dashboard shows the number of active Before-checkout rules and "Checkout protection active", with no counts. | An honest message instead of an invented number. |
| v1.1 | **Shadow evaluation in the worker:** each new order is also evaluated against the BEFORE rules using the order's facts. When a `hide_cod` rule matches and the order was paid by a non-COD method, record `COD_HIDDEN` with savedKind `NONE` ("buyer converted to prepaid"). | "N buyers switched from COD to prepaid": real, verifiable value. |
| v1.1 (Plus only) | **Checkout reporter UI extension** (`purchase.checkout.payment-method-list.render-after` and `purchase.checkout.block.render`) reads the same `$app` metafield (`[[extensions.metafields]]` in its toml), re-runs the engine with the same facts, and on a match `POST`s `/api/checkout-events` once per checkout (deduped by the hashed checkout token). It needs **network access** approval in the Partner Dashboard. | `CHECKOUT_BLOCKED` / `COD_HIDDEN` rows with masked email, cart total, and an estimated saving after 24 h (12.3). |

---

## 10. After-checkout path

### 10.1 Webhook intake (`routes/webhooks.orders.tsx`)

```ts
export const action = async ({ request }: ActionFunctionArgs) => {
  const { topic, shop, payload } = await authenticate.webhook(request);     // HMAC verified, 401 otherwise
  const shopRow = await db.shop.findUnique({ where: { domain: shop }, select: { id: true, uninstalledAt: true } });
  if (!shopRow || shopRow.uninstalledAt) return new Response(null, { status: 200 });

  const orderId = payload.admin_graphql_api_id as string;
  switch (topic) {
    case 'ORDERS_CREATE':
      await queues.orders.add('orders.evaluate', { shopId: shopRow.id, orderId, attempt: 0 },
        { jobId: `eval:${shopRow.id}:${orderId}` });                         // dedupes Shopify retries
      break;
    case 'ORDERS_CANCELLED':
      await queues.orders.add('orders.cancelled', { shopId: shopRow.id, orderId },
        { jobId: `cancelled:${shopRow.id}:${orderId}` });
      break;
  }
  return new Response(null, { status: 200 });
};
```

`jobId` protects against duplicates only while the job is retained, so `removeOnComplete` is kept for 24 h (`{ age: 86400 }`). The **durable** guard is the `MatchLog @@unique([shopId, stage, orderId])` combined with per-action state (10.5).

### 10.2 Facts collection (`facts.server.ts`)

**Order query** (one call; 🔎 verify every field against the pinned version):

```graphql
query OrderFacts($id: ID!) {
  order(id: $id) {
    id name createdAt test cancelledAt sourceName
    displayFinancialStatus displayFulfillmentStatus
    totalPriceSet { shopMoney { amount currencyCode } }
    email phone clientIp
    paymentGatewayNames
    billingAddress  { countryCodeV2 provinceCode city zip firstName lastName }
    shippingAddress { countryCodeV2 provinceCode city zip firstName lastName }
    customer { id }
    risk { assessments { riskLevel provider { id } } }
    shopifyProtect { status eligibility { status } }
    fulfillmentOrders(first: 20) { nodes { id status } }
  }
}
```

**Facts mapping:**

| Fact | Source | Notes |
| --- | --- | --- |
| `orderTotal` | `totalPriceSet.shopMoney.amount` | Always shop currency. |
| `email` | `order.email` → domain; `disposable` from `EmailChecker` | |
| `phone` | `order.phone` or shipping phone → E.164; `status` from `PhoneChecker` (v1.1) | |
| `ip` | `clientIp`; country and proxy from `IpChecker` (v1.1) | |
| `chargebacks` | `Chargeback` rows matching `customerId` OR `emailHash` OR `phoneHash`, counted per window from `initiatedAt` | One SQL query with `FILTER` clauses. Counted only if at least one identity exists. |
| `riskLevel` | max level of assessments with `provider == null` | `PENDING` → defer. |
| `billing`, `shipping` | addresses | |
| `protect` | `shopifyProtect.eligibility.status` | null → unknown (not on Shopify Payments). |

**Enrichment is lazy:** only call an adapter if an enabled AFTER rule uses that filter with an operator that needs it (`email.is_disposable`, `phone.status_*`, `ip.country_*` / `is_proxy`). Run the needed adapters in parallel with `Promise.allSettled`.

### 10.3 Evaluate job (`jobs/handlers/orders.evaluate.ts`)

```
1  load shop; if uninstalled → done
2  rules = enabled AFTER rules (validated; invalid ones skipped + Sentry)
   if rules empty → done (no usage increment)
3  if MatchLog exists for (shop, AFTER, orderId) → resume executor for PENDING actions (10.5) → done
4  order = OrderFacts query
   if order.cancelledAt                       → done
   if sourceName ∈ settings.excludedSources   → done
5  usage: increment ordersEvaluated (only on attempt 0); if over plan quota → done + flag shop (13)
6  facts = collectFacts(order, rules)
7  if any rule uses risk_level AND risk is PENDING/missing AND attempt < 3:
       re-enqueue same job with delay [30s, 120s, 300s][attempt], attempt+1, jobId `eval:{shop}:{order}:d{attempt+1}`
       → done
8  { matched, inconclusive } = evaluate(rules, facts, { unknownAsMatch })
   if no matched and no inconclusive → done (no row)
9  plan = buildPlan(matched)
   status = settings.dryRun ? DRY_RUN : matched.length ? PROCESSING : SKIPPED ("data unavailable")
   INSERT MatchLog (ON CONFLICT (shopId, stage, orderId) DO NOTHING) with evaluation, actions[] = plan as PENDING
       savedKind = dryRun && plan.cancel ? WOULD_HAVE : NONE ; amountSaved = WOULD_HAVE ? orderTotal : 0
   if conflict → go to step 3 behaviour
10 if status == PROCESSING → executor.run(log)   (10.5)
```

Job options: `attempts: 5`, `backoff: { type: 'exponential', delay: 5000 }`. When the final attempt fails: set the row's `status = FAILED` (if a row exists), record `error`, notify the shop (if `notifyOnFailure`) and alert Sentry.

### 10.4 Action preconditions

The executor checks preconditions against **fresh** order state, re-queried before the first mutation (`cancelledAt`, `displayFulfillmentStatus`, `fulfillmentOrders.status`):

| Action | Applicable when | Otherwise |
| --- | --- | --- |
| `tag` | Order not cancelled | `SKIPPED` "Order already cancelled" |
| `hold` (in review) | ≥ 1 fulfillment order with status `OPEN` or `SCHEDULED` | `SKIPPED` "Nothing left to fulfil" |
| `cancel` | Not cancelled and `displayFulfillmentStatus ∈ {UNFULFILLED}` | `SKIPPED` "Order already fulfilled / cancelled" |

### 10.5 Executor (`executor.server.ts`)

Actions run **in a fixed order: tag → hold → cancel**. Tagging first means the order is visibly flagged even if a later step fails. Each `ActionState` is persisted before and after its mutation:

```ts
export async function runExecutor(admin: AdminApi, log: MatchLogRow) {
  for (const a of orderBy(log.actions, ['tag', 'hold', 'cancel'])) {
    if (a.state !== 'PENDING') continue;                        // idempotent resume
    try {
      const pre = await checkPrecondition(admin, log.orderId!, a);
      if (!pre.ok) { await setAction(log.id, a.type, { state: 'SKIPPED', reason: pre.reason }); continue; }
      const ref = await perform(admin, log.orderId!, a);        // throws on userErrors / network
      await setAction(log.id, a.type, { state: 'DONE', ref });
    } catch (e) {
      if (isRetryable(e) && a.attempts < 4) { await bumpAttempts(log.id, a.type); throw e; }  // job retries
      await setAction(log.id, a.type, { state: 'FAILED', reason: humanize(e) });
    }
  }
  await finalizeStatus(log.id);    // derive status, amountSaved, savedKind; append timeline
}
```

**Mutations:**

| Action | Mutation | Details |
| --- | --- | --- |
| tag | `tagsAdd(id: $orderId, tags: $tags)` | `ref.tagsAdded` = only the tags that weren't already on the order (so undo never removes the merchant's own tags). |
| hold | `fulfillmentOrderHold(id, fulfillmentHold: { reason: OTHER, reasonNotes: "Sentinova: in review", notifyMerchant: false, handle: "sentinova" })` for each open FO | Store `ref.holdIds`. 🔎 Verify the `handle` argument and the returned hold id (spike S5). Then `tagsAdd(reviewTag)`. |
| cancel | `orderCancel(orderId, reason, notifyCustomer, restock, staffNote, refundMethod: { originalPaymentMethodsRefund: true } )` | Use `refund: Boolean` instead if the pinned version predates `refundMethod` (🔎 S4). The mutation returns a `job`. Store `ref.jobId`. The final confirmation arrives via `orders/cancelled` (10.7). If `refund=false` or the order has no captured payment (COD), the refund part is a no-op. |

**Retryable** errors are network errors, HTTP 5xx, and GraphQL `THROTTLED`, retried after the `throttleStatus` restore time. Everything else (`userErrors`) is final. `humanize()` maps known codes to merchant text, for example: "Payment gateway can't refund automatically. Refund manually in Shopify admin."

**`finalizeStatus` derivation:**

| Condition | status | savedKind / amountSaved |
| --- | --- | --- |
| any action FAILED | `FAILED` | if cancel DONE → ACTUAL / total, else NONE |
| cancel DONE | `CANCELLED` | ACTUAL / orderTotal |
| hold DONE | `IN_REVIEW` | NONE (until rejected) |
| tag DONE only | `TAGGED` | NONE |
| all SKIPPED | `SKIPPED` | NONE |

### 10.6 Decisions: Approve, Reject, Undo (`decisions.server.ts`)

These run inline in the route action (they're fast), with the same executor primitives and per-action idempotency. The actor is the staff user id from the session token's `sub` claim (with name and email from the online session if available).

| Decision | Allowed from | Steps | Result |
| --- | --- | --- | --- |
| **Approve** | `IN_REVIEW`, `FAILED` with a DONE hold | `fulfillmentOrderReleaseHold(id, holdIds)` for each stored hold → `tagsRemove(reviewTag)` → `tagsAdd(approvedTag)` | `APPROVED`, saved NONE |
| **Reject** | `IN_REVIEW` | Release holds (🔎 S5: only if `orderCancel` refuses held FOs) → `orderCancel` with options from a confirmation modal (defaults: FRAUD, notify on, refund on, restock on) → `tagsRemove(reviewTag)` | `CANCELLED`, ACTUAL / orderTotal |
| **Undo** | `TAGGED`, `IN_REVIEW`, `DRY_RUN` (no-op) | `tagsRemove(ref.tagsAdded)`; release holds if any | `REVERTED`, saved NONE |

Cancel can't be undone. The UI doesn't offer it. Each decision appends a timeline event and sets `decidedBy` and `decidedAt`. Double-clicks are safe: the status precondition is checked in `UPDATE … WHERE status = 'IN_REVIEW' RETURNING`, so only one request wins.

### 10.7 Reconciling external changes

| Webhook | Handling |
| --- | --- |
| `orders/cancelled` | Find the AFTER `MatchLog` for the order. If the cancel action is PENDING or DONE → mark DONE and confirm the timeline. If status is `IN_REVIEW` (merchant cancelled in the Shopify admin) → `CANCELLED`, ACTUAL saved, timeline `CANCELLED_EXTERNALLY`. |
| `fulfillment_orders/hold_released` | If the released hold id is in a log's `ref.holdIds` and status is `IN_REVIEW` → `APPROVED`, actor `system`, timeline `HOLD_RELEASED_EXTERNALLY`. Echoes of our own release are no-ops because the status has already changed. |

---

## 11. Chargebacks (`chargebacks.server.ts`)

1. **Webhook `disputes/create|update`** → job `disputes.sync` (jobId `dispute:{disputeId}:{status}`).
2. The job fetches the dispute's order (`customer { id }`, `email`, `phone`), computes hashes, and upserts `Chargeback` on `(shopId, disputeId)`. Only disputes with `type = chargeback` are counted (inquiries are ignored). Every status counts, won or lost, because the signal is "this buyer filed a chargeback".
3. If the order has a customer, recompute that customer's window counts and write the customer metafield `$app:sentinova.chargebacks = {"d30":0,"d90":1,"d365":1,"all":2,"asOf":"2026-10-16"}` via `metafieldsSet`.
4. **Nightly `chargebacks.refresh-metafields`:** for each customer with a chargeback whose `initiatedAt` is between 29 and 366 days ago, recompute and rewrite the metafield, so windowed counts decay without the Function needing a clock.
5. **Backfill on install:** `disputes.backfill` pages through `shopifyPaymentsAccount { disputes(first: 50, after:) }` (🔎 S8). It does nothing for shops without Shopify Payments.
6. **Manual import (v1.1):** a CSV upload on the Settings page (`email, phone, amount, currency, date`) creates `source = MANUAL` rows. Raw values are hashed and discarded immediately.

---

## 12. Analytics (`analytics.server.ts`)

All dashboard numbers come from `MatchLog`, computed per request. `$tz` = `Shop.timezone`. The range is in the shop's local days.

### 12.1 Queries

```sql
-- KPI tiles + breakdown by status, one pass
SELECT status, count(*) AS n, coalesce(sum("amountSaved") FILTER (WHERE "savedKind" IN ('ACTUAL','ESTIMATED')), 0) AS saved,
       coalesce(sum("amountSaved") FILTER (WHERE "savedKind" = 'WOULD_HAVE'), 0) AS would_have
FROM "MatchLog"
WHERE "shopId" = $1 AND "createdAt" >= $2 AND "createdAt" < $3
GROUP BY status;

-- Trend (daily buckets in shop timezone)
SELECT date_trunc('day', "createdAt" AT TIME ZONE $4) AS day,
       count(*) FILTER (WHERE status NOT IN ('SKIPPED','DRY_RUN')) AS actioned,
       sum("amountSaved") FILTER (WHERE "savedKind" IN ('ACTUAL','ESTIMATED')) AS saved
FROM "MatchLog" WHERE "shopId" = $1 AND "createdAt" >= $2 AND "createdAt" < $3
GROUP BY 1 ORDER BY 1;

-- Top triggered rules (also used for "times triggered / last triggered" on the Rules page, range = all)
SELECT rid AS "ruleId", count(*) AS n, max("createdAt") AS last
FROM "MatchLog", unnest("ruleIds") AS rid
WHERE "shopId" = $1 AND "createdAt" >= $2 AND "createdAt" < $3
GROUP BY rid ORDER BY n DESC LIMIT 5;
```

Rule names for deleted rules come from the latest `evaluation.rules[].name`.

### 12.2 Dashboard widgets mapping

| Widget (spec §6.1) | Source |
| --- | --- |
| Live rules | `count(*) FROM Rule WHERE enabled GROUP BY stage` |
| Orders protected | rows with status ∈ {CANCELLED, IN_REVIEW, APPROVED, TAGGED, CHECKOUT_BLOCKED, COD_HIDDEN} |
| Breakdown by action | status counts (COD disabled and Checkout blocked from v1.1) |
| Money saved | `saved` (ACTUAL + ESTIMATED), with ESTIMATED shown as a separate "incl. X estimated" line. In dry-run, show `would_have` as "Would have saved". |
| Trend chart | daily query |
| Top rules | top rules query |
| False-positive rate (internal metric, spec §13) | `APPROVED / (APPROVED + CANCELLED where decidedBy not null)` |

### 12.3 Money-saved rules

| Event | savedKind | Amount |
| --- | --- | --- |
| Auto-cancel DONE | ACTUAL | order total (shop currency, at evaluation time) |
| Rejected from review / cancelled externally while in review | ACTUAL | order total |
| Approved, undone, tag only, skipped | NONE | 0 |
| Dry-run row whose plan would cancel | WOULD_HAVE | order total (never added to "saved") |
| Checkout event (v1.1) | PENDING_ESTIMATE → ESTIMATED or NONE | cart total in shop currency. The hourly `analytics.finalize-estimates` job promotes rows older than 24 h to ESTIMATED **only if** no order with the same `emailHash` was created in the window; otherwise it sets NONE. |

Amounts are stored with the currency at the time of the event. If a shop changes currency (rare), the dashboard sums per currency and shows the current currency with a footnote.

---

## 13. Billing (`billing.server.ts`)

- Use **Shopify Managed Pricing**: plans are defined in the Partner Dashboard and Shopify hosts the plan-selection page, so there's no `appSubscriptionCreate` code. Read the active plan from `currentAppInstallation { activeSubscriptions { name status } }` on install and on `app_subscriptions/update`, and map it to `Shop.billingPlan`.
- **Limits live in one file** (`app/lib/plans.ts`). The numbers below are placeholders until pricing is decided (spec §12):

| Plan | Enabled rules | Orders evaluated / 30 days | Before-checkout | Enrichment (phone/IP) |
| --- | --- | --- | --- | --- |
| FREE | 3 | 100 | ✓ | ✗ |
| GROWTH | 15 | 2,000 | ✓ | ✓ (fair use) |
| PRO | 50 | 20,000 | ✓ | ✓ |

- **Usage counter:** in `orders.evaluate` step 5, `UPDATE "Shop" SET "ordersEvaluated" = CASE WHEN now() - "usagePeriodStart" > interval '30 days' THEN 1 ELSE "ordersEvaluated" + 1 END, "usagePeriodStart" = CASE … END RETURNING …`. This is atomic, needs no extra table, and only counts shops that have enabled AFTER rules.
- **Over quota:** stop evaluating, show a dashboard banner with an upgrade link, and email the owner once per period. Before-checkout keeps working (it costs us nothing).
- **Downgrade or lapse:** keep all data, and disable the lowest-priority rules beyond the limit (`enabled = false`, show a notice). Never delete rules.
- Dev stores (`partnerDevelopment`) get PRO limits for free.

---

## 14. Background jobs

All queues live in BullMQ on one Redis instance, with a global worker concurrency of 10. Repeatable jobs are registered once at worker start (`upsertJobScheduler`).

| Job | Queue | Trigger | jobId (dedupe) | Retries | Handler summary |
| --- | --- | --- | --- | --- | --- |
| `orders.evaluate` | orders | `orders/create`, deferral | `eval:{shop}:{order}[:dN]` | 5, exp 5 s | 10.3 |
| `orders.cancelled` | orders | `orders/cancelled` | `cancelled:{shop}:{order}` | 5 | 10.7 |
| `fo.hold-released` | orders | webhook | `holdrel:{holdId}` | 5 | 10.7 |
| `disputes.sync` | disputes | `disputes/*` | `dispute:{id}:{status}` | 5 | 11 |
| `disputes.backfill` | disputes | install | `backfill:{shop}` | 3 | 11 |
| `snapshot.sync` | sync | sync failure, list refresh | `snapshot:{shop}` | 8, exp 10 s | 9.2 |
| `snapshot.reconcile` | sync | hourly | scheduler | – | 9.2 |
| `chargebacks.refresh-metafields` | maintenance | daily 02:00 UTC | scheduler | 3 | 11 |
| `disposable.refresh` | maintenance | daily 03:00 UTC | scheduler | 3 | Download the list, validate (≥ 1,000 entries, ≤ 20% change from the previous version, otherwise keep the old list and alert), store in Redis set `disposable:full` and the bloom bytes in `disposable:bloom`, then enqueue `snapshot.sync` for active shops with BEFORE email rules |
| `retention.purge` | maintenance | daily 04:00 UTC | scheduler | 3 | Delete `MatchLog` older than `retentionMonths` and `Chargeback` older than 25 months, in batches of 5,000 |
| `analytics.finalize-estimates` | maintenance | hourly (v1.1) | scheduler | 3 | 12.3 |
| `privacy.*` | privacy | compliance webhooks | `privacy:{topic}:{id}` | 5 | 15.4 |

**Per-shop fairness:** a large shop must not starve the others. Use BullMQ's rate limiter keyed by shop (`limiter` with a group key, or a Redis token bucket checked at handler start: max 2 concurrent Admin API calls per shop). On `THROTTLED`, re-queue with a delay equal to the restore time.

**Dead letters:** jobs that exhaust their retries stay in the failed set for 14 days. An admin script (`scripts/replay-failed.ts --job orders.evaluate --since 2h`) replays them. Alert when the failed count is above 20 in 15 min.

---

## 15. Security, privacy and compliance

### 15.1 Authentication and authorization

- Admin routes use `authenticate.admin(request)` (session token / token exchange) in **every** loader and action.
- Webhooks use `authenticate.webhook(request)`, which verifies the HMAC and rejects with 401.
- The checkout events API (v1.1) uses `authenticate.public.checkout(request)`, with a per-shop rate limit of 60/min and a body schema validated by zod.
- Staff access follows Shopify app permissions. There are no in-app roles in v1. The actor is recorded for decisions.

### 15.2 Scope justification (for App Store review)

| Scope | Why |
| --- | --- |
| `write_orders` | Read orders for evaluation; `tagsAdd`/`tagsRemove`; `orderCancel`. |
| `write_customers` | Write the chargeback count to the customer metafield read by the Function; read customer identity on orders. |
| `write_merchant_managed_fulfillment_orders` | Place and release holds for "In review". |
| `write_third_party_fulfillment_orders` | Holds when the order is assigned to a fulfillment service location. 🔎 S5: drop it if holds there aren't possible anyway. |
| `read_shopify_payments_disputes` | Chargeback counts: dispute webhooks and backfill. |
| `write_payment_customizations` | Create the COD-hiding customization. |
| `write_validations` | Create the checkout validation (v1.1). Add it in the v1.1 release to keep MVP scopes minimal. |

**Protected customer data:** request Level 2 access for **email, phone, name and address** in the Partner Dashboard during M0. Purposes: fraud detection (email, phone, address, name), with no marketing use. Without approval, these fields come back null in production stores.

### 15.3 Data handling

| Data | Stored as | Where |
| --- | --- | --- |
| Email | HMAC-SHA256 hash + masked form (`j•••@domain`) | MatchLog, Chargeback |
| Phone | Hash; masked `+234•••••78` inside `evaluation.actual` | MatchLog, Chargeback |
| IP | Masked to /24 (v4) or /48 (v6) inside `evaluation.actual` | MatchLog |
| Name, address | **Not stored.** Mismatch results are stored as booleans per field. | – |
| Customer display name | **Not stored.** Fetched live for the visible Orders page via `nodes(ids:)`. | – |
| Enrichment cache | Key = `hash(input)`, value = result only | Redis, TTL: email 7 d, phone 30 d, IP 24 h |

Hashing: `hmacSha256(HASH_PEPPER, normalize(value))`, hex. Searching the Orders page by email hashes the search term and matches `emailHash`.

Third-party providers get the minimum: the phone provider gets the E.164 number only, the IP provider the IP only, and the email list is local, so no email ever leaves the system.

### 15.4 Compliance webhooks (`privacy.server.ts`)

Each handler responds 200 immediately and processes in a job.

| Topic | Action |
| --- | --- |
| `customers/data_request` | Gather the `MatchLog` and `Chargeback` rows for `customer.id`, the email hash and the phone hash, render a JSON summary (statuses, dates, rules, masked identifiers), and email it to the shop owner's contact email within 30 days, logging the request. |
| `customers/redact` | Delete `Chargeback` rows matching customer id, email hash or phone hash. For `MatchLog` rows of `orders_to_redact` or the customer: null out `customerId`, `emailHash`, `phoneHash` and `maskedEmail`, and scrub masked values from `evaluation.actual`. The rows themselves stay, as anonymous statistics. Then recompute the customer metafield (or delete it). |
| `shop/redact` | `DELETE FROM "Shop" WHERE domain = $1`, which cascades to all tables, and delete `Session` rows. |

### 15.5 Other requirements

- Secrets live in the host's secret store; nothing secret goes in the repo. Postgres is encrypted at rest (provider default) and uses TLS in transit.
- Rule values are user input. They're zod-validated, length-bounded, and rendered only as text, never as HTML.
- Security headers are handled by the template (CSP `frame-ancestors` for the admin).
- The privacy policy page lists the providers, purposes, retention and contact.

---

## 16. Admin UI

Stack: React Router routes with Polaris web components (`<s-page>`, `<s-section>`, `<s-table>` …) inside App Bridge. Data is loaded in `loader`s and mutated in `action`s. Every list's state (filters, pagination, search) lives in **URL search params** so views can be shared and the back button works.

### 16.1 Navigation

`app.tsx` renders the `<s-app-nav>`: **Dashboard**, **Rules**, **Orders**, **Settings**. If `snapshotError` is set, the app shows a global critical banner. If the shop is over quota, it shows a warning banner. If `dryRun` is on, it shows an info banner ("Log-only mode: no orders are being changed. Go live").

### 16.2 Dashboard (`/app`)

- **Loader:** `range` param (`today|7d|30d|90d|custom&from&to`, default `30d`) → analytics queries (12.1) run in parallel, plus rule counts and onboarding state.
- **Layout:** KPI row (Live rules split Before/After · Orders protected · Money saved, with an "estimated" footnote), breakdown by action, trend chart (two series, daily), and a top 5 rules table that links to each rule.
- **Empty state** (no rules): an onboarding card with "Create your first rule" plus the three templates.
- **Performance:** p95 < 300 ms server time.

### 16.3 Rules list (`/app/rules`)

- **Loader:** rules for the shop, grouped by stage (tabs: Before checkout / After checkout), filters `status=on|off`, `q=name`. Times triggered and last triggered come from the top rules query with the range set to all time.
- **Columns:** drag handle + priority, name, conditions summary (e.g. "Order total > 200 **and** Email is disposable"), actions summary, On/Off toggle, times triggered, last triggered, and a row menu (Edit, Duplicate, Move up, Move down, Delete).
- **Actions** (one `action` with an `intent` field):
  - `toggle {id, enabled}`, checked against plan limits when enabling.
  - `reorder {stage, orderedIds[]}`: a full renumber in a transaction (6.2). Drag-and-drop and Move up/down both send the full order.
  - `duplicate {id}` → copy named "<name> (copy)", disabled, placed at the bottom of the stage.
  - `delete {id}` → confirmation modal first.
  - After any BEFORE-stage change → `syncSnapshot` (9.2).
- **Optimistic UI** for toggles and reorder (`useFetcher`). The app rolls back and shows a toast on error.

### 16.4 Rule editor (`/app/rules/new`, `/app/rules/:id`)

- **Tab 1 – Conditions:** name, stage (radio; changing stage clears filters and actions that aren't valid for the new stage, after a confirmation), match type, and condition rows (Filter → Operator → Value), all driven by `FILTERS` from the registry. Value inputs by kind: `money` (with the shop currency suffix), `number`, `number+window`, `enum`, `multiselect`, `text`, `list` (textarea, one entry per line, with counts), `fields` (checkbox group), `none`. Filters not yet released are hidden. Stage-specific hints come from the registry (e.g. "Before checkout: only logged-in customers").
- **Tab 2 – Actions:** the action section for the chosen stage; the "Stop processing further rules" toggle; and a live summary: *"If order total > 200 and email is disposable, then hide Cash on Delivery."*
- **Save:** the action validates with the **same zod schema** used client-side (via the engine's schema entry) and returns field-level errors keyed by `conditions.2.value.amount`, which the UI shows inline. A new rule gets `priority = max + 1` in its stage. On edit, an optimistic lock (`WHERE version = $v`) returns 409 with "This rule was changed by someone else. Reload."
- **Leaving with unsaved changes:** App Bridge `SaveBar` (`<ui-save-bar>`).

### 16.5 Orders (`/app/orders`)

- **Loader params:** `status`, `ruleId`, `stage`, `from`, `to`, `q` (an order name like `#1001`, or an email, which is hashed), and `cursor` (`createdAt|id`, page size 50). For the visible page, one `nodes(ids:)` call fetches `name`, `customer.displayName` and `displayFinancialStatus`, with a 5 s timeout. If it fails, the column shows "—".
- **Columns:** order (links to `shopify:admin/orders/{numericId}`), date (shop timezone), customer (live name + masked email), total, rule(s), action(s) (chips from `actions[].state`), status badge, amount saved (with an "est." suffix when ESTIMATED).
- **Detail drawer** (`<s-modal>` or side panel), loaded via fetcher from `/app/orders/:id`: the matched rules with each condition rendered as "Order total **450.00** > 300.00 ✓", one line per action state with its reason, the timeline, and errors with remediation text.
- **Buttons:** Approve, Reject (opens a confirmation modal with the cancel options) and Undo, shown per the status rules in 10.6. Each POSTs to `/app/orders/:id` with an `intent`.
- **CSV export:** `/app/orders/export?…same filters` streams up to 50,000 rows with these columns: order name, date, masked email, total, currency, rules, actions, status, amount saved, saved kind. In the embedded app, fetch it with the session token and save as a Blob (App Bridge doesn't allow top-level navigation downloads with auth).

### 16.6 Settings (`/app/settings`)

The page has sections for **Mode** (dry-run toggle), **Unknown data** (unknownAsMatch, with an explanation), **Checkout** (COD method names, block message, protection status and plan compatibility), **Tags** (review and approved tags), **Order sources** (excluded sources), **Data** (retention months), **Notifications** (on failure) and, from v1.1, **Chargebacks** (CSV import). Saving validates with `ShopSettings`, and snapshot-relevant changes trigger `syncSnapshot`.

### 16.7 UX non-functionals

- All pages are usable at a 375 px admin width (Shopify mobile) and use Polaris components only, so accessibility comes for free.
- Every async button has a loading state, and every mutation ends with a toast.
- Money is formatted with `Intl.NumberFormat(shop locale, { currency })`, and dates use the shop timezone.

---

## 17. Testing strategy

| Layer | Tooling | Must cover | Gate |
| --- | --- | --- | --- |
| Rule engine | Vitest | Every filter × operator incl. unknown; Kleene ALL/ANY; priority + stopOnMatch; plan merge (7.6); normalisers; CIDR v4/v6; bloom false-positive rate on a 10k random set ≤ 0.2%; snapshot encode/decode round trip and size guard | ≥ 95% line coverage |
| Golden parity | Vitest + `shopify app function run` | The same fixtures across engine, Function and worker (7.8) | 100% pass |
| Functions | `shopify app function run` | hide on match; no-op on dry-run, missing metafield, bad JSON, unknown `v`, no COD method, guest without email; presentment currency conversion | Instruction and size budget (9.5) |
| Services | Vitest + mocked Admin GraphQL (msw) + test Postgres (Docker) | Idempotency (same webhook twice, crash between actions then resume); partial failure; THROTTLED retry; deferral path; decisions concurrency (two approves); tenant isolation (cross-shop id → 404) | Pass |
| Webhooks | Integration | HMAC rejection (401); 200 in < 1 s; uninstalled shop no-op | Pass |
| E2E (dev stores) | Manual script plus Playwright for the admin UI | Scenarios below | Before each release |
| Load | `scripts/replay-webhooks.ts` | 1,000 `orders/create` in 60 s across 5 shops: no duplicate actions, p95 evaluate latency < 10 s, no per-shop starvation | Before launch |

**Scenarios to automate or script:**

1. A high-value COD cart with a disposable email → COD hidden; with a gmail address → COD visible.
2. A guest who hasn't entered an email → COD visible (unknown is never a match before checkout).
3. HIGH-risk order → tagged and held → Approve releases the hold and swaps the tag.
4. Risk PENDING at create → deferred → evaluated when the risk arrives.
5. A customer with 2 chargebacks → cancelled, refunded and restocked; the metafield is updated; the next checkout hides COD.
6. Phone provider timeout → condition unknown → `SKIPPED` "data unavailable" row; with `unknownAsMatch` on → actioned.
7. Duplicate `orders/create` → one row, one set of mutations.
8. Dry-run → `DRY_RUN` row, "would have saved", zero mutations (assert with the msw call log).
9. Merchant cancels an in-review order in the Shopify admin → status `CANCELLED`, saved counted.
10. Uninstall → reinstall within 48 h → rules intact, snapshot re-synced, Function re-activated.
11. `customers/redact` → chargebacks deleted, logs anonymised.

---

## 18. Deployment and operations

### 18.1 Environments

| Env | Shopify app | Infra | Deploy |
| --- | --- | --- | --- |
| local | Personal dev app (`shopify app dev`) | Docker Postgres and Redis | – |
| staging | `Sentinova (staging)` app, `shopify.app.staging.toml` | Separate DB and Redis | Automatically on merge to `main` |
| production | `Sentinova - Spam Protection` | Managed Postgres (daily backups + PITR), managed Redis | Manual promote of a tagged release |

### 18.2 CI (GitHub Actions)

Each PR runs: `pnpm install --frozen-lockfile` → lint (ESLint + Prettier) → `tsc --noEmit` → GraphQL codegen check (no diff) → unit and golden tests → services tests with Postgres and Redis service containers → `prisma migrate diff` (it fails if the schema changed without a migration) → Function build + instruction and size budget check → app build.

### 18.3 Release procedure

1. `prisma migrate deploy` (migrations MUST be backward-compatible with the running code: add → backfill → switch → remove, across two releases).
2. Deploy web and worker (worker first when job payloads change; job handlers MUST accept the previous payload version).
3. `shopify app deploy` for extension changes. Functions MUST accept every snapshot `v` the app can currently write, and the app MUST NOT write a new `v` until the Function version that reads it is live (deploy the Function first, then the app).

### 18.4 Observability

- **Logs:** pino JSON with `shop`, `orderId`, `jobId` and `requestId` on every line. No PII in logs (a lint rule bans logging `email`, `phone` and `ip` fields).
- **Errors:** Sentry for web and worker, tagged with shop.
- **Metrics:** queue depth and age per queue, job latency (p50/p95), action failure rate, provider latency and error rate, circuit-breaker state, snapshot sync failures, webhook 2xx rate.
- **Alerts:** queue age > 5 min; action failure rate > 5% over 15 min; provider error rate > 20%; more than 10 snapshot sync failures per hour; failed jobs > 20 per 15 min.

### 18.5 Runbooks (write before launch)

| Situation | First steps |
| --- | --- |
| Provider outage | The circuit breaker opens automatically, so conditions become unknown. Confirm on the dashboard, consider a temporary `unknownAsMatch` advisory to affected merchants, and keep the status page updated. |
| Queue backlog | Scale the worker; check for a THROTTLED storm from a single shop; pause that shop's jobs if needed. |
| Bad rule mass-cancelled orders | Help the merchant disable the rule; export affected rows (`status=CANCELLED&ruleId=`); orders can't be uncancelled, so guide them to re-create orders or draft orders. |
| Function misbehaving | Set `dry: true` in the snapshot via the admin script for affected shops; roll back the extension version in the Partner Dashboard. |
| Hash pepper leak | Rotate: deploy dual-pepper matching, rehash rows from Shopify data where possible, and drop the old pepper. |

---

## 19. Milestone plan

The estimates assume one developer working part-time. Each milestone ends with its exit criteria demonstrated on a dev store.

| Milestone | Scope | Exit criteria |
| --- | --- | --- |
| **M0 – Foundations & spikes** (wk 1–2) | Scaffold, workspace, Prisma schema, CI, staging app, protected-data request submitted, **spikes S1–S8 (section 21)** | App installs and loads; spike results recorded in `docs/spikes.md`; the 🔎 items in this guide resolved or given a fallback |
| **M1 – Rule engine** (wk 3) | `packages/rule-engine` complete for MVP filters, plan merge, schema, snapshot, bloom, golden fixtures | ≥ 95% coverage; golden suite green |
| **M2 – Rules UI** (wk 4–5) | Rules list, editor, reorder, duplicate, toggle, validation, plan limits, Settings page | Rules CRUD works end to end; optimistic lock tested |
| **M3 – After-checkout pipeline** (wk 6–7) | Webhooks, queues, facts, deferral, evaluation, MatchLog, executor (tag, hold, cancel incl. restock), dry-run, reconciliation webhooks, chargebacks + backfill | Scenarios 3–5, 7–9 pass on a dev store |
| **M4 – Orders page & decisions** (wk 8) | Orders table, drawer, Approve/Reject/Undo, live customer names | Full in-review lifecycle verified |
| **M5 – Before-checkout** (wk 9–10) | Snapshot sync + reconcile, COD Function, install activation, onboarding | Scenarios 1–2, 10 pass; budgets met; fail-open verified |
| **M6 – Dashboard & billing** (wk 11) | Analytics queries, dashboard, managed pricing, usage counter | Dashboard numbers match a hand-computed fixture |
| **M7 – Hardening & launch** (wk 12–13) | Compliance webhooks, retention, load and security tests, runbooks, listing assets, review checklist | **MVP submitted to the App Store** |
| **M8 – v1.1** (wk 14–17) | Phone and IP adapters + caching, checkout validation (Pay now), shadow evaluation, checkout reporter (Plus), estimates job, CSV export, manual chargeback import, trend refinements | v1.1 scenarios pass; `write_validations` scope added via an app version update |

---

## 20. App Store readiness checklist

- [ ] Protected customer data access approved (email, phone, name, address)
- [ ] Compliance webhooks implemented, tested and documented
- [ ] Scopes minimal and justified (15.2)
- [ ] Managed pricing plans configured; free plan works without a card
- [ ] Onboarding works on a fresh store; app usable within 2 minutes of install
- [ ] Embedded pages load in < 2 s with no console errors, and Web Vitals pass the Shopify "Built for Shopify" thresholds
- [ ] Before-checkout behaviour degrades clearly on plans that don't support it
- [ ] Uninstall/reinstall verified (8.3)
- [ ] Privacy policy, support email and FAQ published
- [ ] Listing: name, tagline, screenshots, demo video, feature list
- [ ] Reviewer instructions and a seeded test store (sample rules, sample flagged orders)

---

## 21. M0 verification spikes

Each spike gets a time box of ≤ 1 day and records **result, evidence (query/screenshot) and decision** in `docs/spikes.md`.

| ID | Question | How to test | Fallback if "no" |
| --- | --- | --- | --- |
| S1 | Are payment customization and checkout validation Functions available to **public apps on non-Plus plans**? | Install on a Basic dev store, create both customizations, place a checkout | Mark Before-checkout actions "Plus only" in the UI and onboarding; the After path is unaffected |
| S2 | Function input: are `shop.metafield` (`$app`), `buyerIdentity.email/phone`, `customer.metafield` and `presentmentCurrencyRate` available at the payment step? What are the metafield and input size limits? | Function that echoes its input in `shopify app function run` and `app dev` logs | Use the `paymentCustomization.metafield` owner instead of shop; treat currency mismatch as unknown; shrink the budget |
| S3 | JS Function instruction count for the worst-case snapshot | Build the worst-case fixture and run it | Reduce limits (rules/lists), precompute more in the snapshot, or port the hot path to Rust |
| S4 | `orderCancel` arguments (`refund` vs `refundMethod`), behaviour on COD/unpaid orders, async job semantics | Cancel test orders on both dev stores | Use the version-appropriate argument; for gateways that can't refund, mark FAILED with instructions |
| S5 | `fulfillmentOrderHold` with `handle`; can held orders be cancelled; 3PL-assigned FOs | Hold, then cancel, on a store with a fulfillment-service location | Release before cancel; drop the 3PL scope if holds there are impossible and report SKIPPED |
| S6 | Can the app list manual payment method names? | Inspect the Admin API for payment settings | Manual entry (current design) |
| S7 | Are app metafields and customizations removed on uninstall? | Uninstall, inspect via another app or GraphiQL | Tolerate leftovers on reinstall (current design) |
| S8 | Field availability: `Order.clientIp`, `Order.risk.assessments.provider`, `Order.shopifyProtect`, dispute `type`, and `shopifyPaymentsAccount.disputes` | GraphiQL on the pinned version | Adjust the facts mapping; mark the filter unavailable where the data doesn't exist |

---

## 22. Coding conventions

- TypeScript `strict`, ESM, no `any` outside generated code (ESLint `no-explicit-any: error`).
- zod at every boundary (request bodies, webhook payload fields used, JSON columns, provider responses).
- All Admin GraphQL documents live in `app/graphql/*.graphql` and are typed with codegen. No inline untyped queries.
- Server-only modules end with `.server.ts`.
- Errors: throw typed errors (`RetryableError`, `UserFacingError`, `SnapshotTooLarge`). Route actions turn them into `{ errors }` responses; jobs use them to decide whether to retry.
- Time is always UTC in the DB, converted to shop time only for display and dashboard bucketing.
- Commits follow Conventional Commits; PRs need green CI, a passing golden suite and an updated guide section when behaviour changes.
