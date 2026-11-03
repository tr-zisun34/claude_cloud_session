# Stagepay — Development Guide

**App:** Stagepay · **App Store listing:** *Stagepay: Milestone Payments*
Version 1.1 · 2026-11-03 · Matches spec v1.2 (reminders on the Dashboard; collection method and days before overdue per rule)

This guide is the hands-on companion to **`docs/deposit-app-spec.md` (v1.1)**. The spec says **what** to build; this guide says **how**, step by step, in the agreed build order (0 → 13). Each step is built, tested and signed off before the next one starts.

> **Naming:** the spec was written before the app had a name. Wherever the spec says `deposit` in a technical identifier, use **`stagepay`** instead (see [Naming](#3-naming)). Behaviour is unchanged.

---

## Contents

1. [How to use this guide](#1-how-to-use-this-guide)
2. [Prerequisites & environment](#2-prerequisites--environment)
3. [Naming](#3-naming)
4. [Project layout](#4-project-layout)
5. [Step 0 — Foundations](#step-0--foundations-14-h)
6. [Step 1 — Deposit rules (P3, P4)](#step-1--deposit-rules-p3-p4-20-h)
7. [Step 2 — Theme app extension (E1)](#step-2--theme-app-extension-e1-13-h)
8. [Step 3 — Order sync & status engine](#step-3--order-sync--status-engine-20-h)
9. [Step 4 — Dashboard, simple (P1 part)](#step-4--dashboard-simple-p1-part-5-h)
10. [Step 5 — Order detail & balance collection (P2)](#step-5--order-detail--balance-collection-p2-28-h)
11. [Step 6 — Emails & reminders](#step-6--emails--reminders-13-h)
12. [Step 7 — Settings (P5)](#step-7--settings-p5-3-h)
13. [Step 8 — Email templates editor (P6)](#step-8--email-templates-editor-p6-3-h)
14. [Step 9 — Dashboard, full (P1 finish)](#step-9--dashboard-full-p1-finish-7-h)
15. [Step 10 — Plan & billing (P7)](#step-10--plan--billing-p7-5-h)
16. [Step 11 — Checkout validation Function (E2)](#step-11--checkout-validation-function-e2-4-h)
17. [Step 12 — Privacy, uninstall, activity log](#step-12--privacy-uninstall-activity-log-7-h)
18. [Step 13 — Full testing & launch](#step-13--full-testing--launch-29-h)
19. [Appendix A — Command cheat sheet](#appendix-a--command-cheat-sheet)
20. [Appendix B — Definition of done (every step)](#appendix-b--definition-of-done-every-step)
21. [Appendix C — Git workflow](#appendix-c--git-workflow)

---

## 1. How to use this guide

- Work through the steps **in order**. Each step lists: **Goal · Spec reference · Builds (pages/extensions) · Tasks · Tests · Manual check · Done checklist**.
- Code blocks are **skeletons**: they show structure, names and the Shopify calls. Fill in validation, error handling and UI details from the spec.
- Lines marked **⚠ Verify** depend on Shopify API details that may differ in your pinned API version. Check them in GraphiQL (`shopify app dev` → press `g`) before relying on them.
- Hours are base estimates from *Build_Order_Estimate.xlsx* (add the 20% buffer for planning).

| Step | Build | Hours | Milestone after this step |
|---|---|---|---|
| 0 | Foundations | 14 | |
| 1 | Deposit rules P3 + P4 | 20 | |
| 2 | Theme extension E1 | 13 | **Buyers can pay a deposit** |
| 3 | Order sync & status engine | 20 | |
| 4 | Dashboard (simple) | 5 | |
| 5 | Order detail + collection P2 | 28 | **Works end to end (demo)** |
| 6 | Emails & reminders | 13 | **No manual chasing** |
| 7 | Settings P5 | 3 | |
| 8 | Email templates P6 | 3 | |
| 9 | Dashboard (full) + Reminders panel | 7 | |
| 10 | Plan & billing P7 | 5 | **Ready to sell** |
| 11 | Validation Function E2 | 4 | |
| 12 | Privacy, uninstall, activity log | 7 | |
| 13 | Testing & launch | 29 | **Ready for App Store review** |
| | **Total** | **171** (205 with buffer) | |

---

## 2. Prerequisites & environment

Assumed already done (not covered here): app created with **Shopify CLI** using the official **React Router** app template (TypeScript, Prisma), app named **Stagepay** in the Partner Dashboard, Git repository, hosting account.

### Tools
| Tool | Version |
|---|---|
| Node.js | 20 LTS or newer |
| Shopify CLI | latest (`npm i -g @shopify/cli@latest`) |
| PostgreSQL | 15+ (local via Docker is fine) |
| Accounts | Shopify Partner, Postmark (or Resend), error tracker (Sentry or similar), Mailtrap (test inbox) |

### Development stores (from the feasibility tests)
| Store | Plan | Payments |
|---|---|---|
| `stagepay-dev-basic` | Non-Plus (Basic/Grow) | Bogus Gateway or Shopify Payments test mode |
| `stagepay-dev-plus` | Plus | Same |

Each store has the **Oak Dining Table** ($1,000) test product and the **Dawn** theme.

### Packages to add

```bash
npm i pg-boss postmark zod decimal.js date-fns date-fns-tz
```
```bash
npm i -D vitest @vitest/coverage-v8 msw @playwright/test
```

### Environment variables (`.env`, never committed)
| Variable | Example | Used by |
|---|---|---|
| `DATABASE_URL` | `postgresql://stagepay:pw@localhost:5432/stagepay` | Prisma, pg-boss |
| `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `SCOPES`, `SHOPIFY_APP_URL` | (set by CLI) | Template |
| `POSTMARK_TOKEN` | `xxxxxxxx` | Emails |
| `EMAIL_FROM_DOMAIN` | `mail.stagepay.app` | Emails |
| `EMAIL_TEST_MODE` | `true` in dev | Sends to Mailtrap / logs only |
| `SENTRY_DSN` | … | Errors |
| `DEV_TOOLS` | `true` in dev only | Hidden `/app/dev` page |
| `TIME_COMPRESSION` | `1440` in dev (1 day = 1 minute) | Reminders and day-based triggers during testing |

### `shopify.app.toml` — scopes and webhooks

```toml
name = "Stagepay"

[access_scopes]
scopes = "read_products,read_purchase_options,write_purchase_options,read_orders,write_orders,read_payment_terms,read_customers,read_payment_mandate,write_payment_mandate,read_fulfillments,write_files"

[webhooks]
api_version = "2026-10"

  [[webhooks.subscriptions]]
  topics = ["app/uninstalled"]
  uri = "/webhooks/app/uninstalled"

  [[webhooks.subscriptions]]
  topics = ["orders/create", "orders/updated", "orders/edited", "orders/cancelled", "order_transactions/create", "refunds/create", "fulfillments/create"]
  uri = "/webhooks/orders"

  [[webhooks.subscriptions]]
  topics = ["shop/update"]
  uri = "/webhooks/shop/update"

  [[webhooks.subscriptions]]
  topics = ["app_subscriptions/update"]
  uri = "/webhooks/app_subscriptions/update"

  [[webhooks.subscriptions]]
  compliance_topics = ["customers/data_request", "customers/redact", "shop/redact"]
  uri = "/webhooks/compliance"
```

⚠ Verify: payment-mandate scopes may need approval in the Partner Dashboard → API access. Request them on day 1 (feasibility test T11).

### Processes
Stagepay runs **two processes** from the same codebase:

| Process | Command | Does |
|---|---|---|
| `web` | `npm run start` (template) | Admin pages, webhooks |
| `worker` | `npm run worker` → `node build/worker.js` | pg-boss jobs (sync, charges, emails, reminders) |

Add to `package.json`: `"worker": "tsx app/worker.ts"` (dev) and a build step for production.

---

## 3. Naming

| Item | Name |
|---|---|
| App / brand | Stagepay |
| Metafield namespace (app-owned) | `stagepay` (keys: `policy`, `minimums`, `schedule`) |
| Selling plan `merchantCode` | `stagepay-<ruleId>` |
| Order tags | `stagepay-deposit-paid`, `stagepay-balance-due`, `stagepay-overdue`, `stagepay-paid` |
| Order note prefix | `[Stagepay]` |
| Theme extension / block | `stagepay-block` / `stagepay-deposit` ("Stagepay deposit") |
| Validation Function | `stagepay-min-order` |
| Job names | `<area>.<action>`, e.g. `rules.sync`, `orders.sync`, `balance.request` |
| Idempotency keys | `stagepay:<stepId>:<attemptNo>` |
| Email From | `"<Shop name>" <notifications@mail.stagepay.app>` |

---

## 4. Project layout

Only Stagepay files are listed; template files stay as generated.

```
app/
  routes/
    app._index.tsx                     P1 Dashboard
    app.orders.$id.tsx                 P2 Order detail
    app.rules._index.tsx               P3 Rules list
    app.rules.$id.tsx                  P4 Create/edit rule ($id = "new" for create)
    app.settings._index.tsx            P5 Settings
    app.settings.emails._index.tsx     P6 Email list
    app.settings.emails.$key.tsx       P6 Email editor
    app.billing.tsx                    P7 Plan & billing
    app.dev.tsx                        Dev tools (DEV_TOOLS only)
    webhooks.orders.tsx
    webhooks.app.uninstalled.tsx
    webhooks.shop.update.tsx
    webhooks.app_subscriptions.update.tsx
    webhooks.compliance.tsx
    webhooks.postmark.tsx
  services/                            server-only business logic (*.server.ts)
    shopify-admin.server.ts
    queue.server.ts
    money.server.ts
    shop.server.ts
    webhook-guard.server.ts
    rules.server.ts
    selling-plans.server.ts
    metafields.server.ts
    order-sync.server.ts
    order-status.server.ts             pure functions (unit-tested)
    balance.server.ts
    email.server.ts
    email-provider.server.ts
    email-defaults.server.ts
    reminders.server.ts
    settings.server.ts
    plan.server.ts
    billing.server.ts
    audit.server.ts
    compliance.server.ts
  jobs/                                one file per job handler
  components/                          UI components
  worker.ts                            starts pg-boss and registers jobs
extensions/
  stagepay-block/                      E1 theme app extension
  stagepay-min-order/                  E2 validation Function
prisma/schema.prisma
tests/
  unit/  integration/  e2e/  fixtures/ (saved webhook payloads from feasibility tests)
```

---

## Step 0 — Foundations (14 h)

**Goal:** shared building blocks every later step uses.
**Spec:** Module 0. **Builds:** no merchant page (hidden `/app/dev` only).

### Tasks

**0.1 Database: `Shop`, `WebhookEvent`** — copy both models from spec Appendix A into `prisma/schema.prisma` (keep the template's `Session` model).

```bash
npx prisma migrate dev --name step0_foundations
```

**0.2 Shopify Admin client wrapper** — one place for GraphQL calls, throttling retry and `userErrors` handling.

```ts
// app/services/shopify-admin.server.ts
import { unauthenticated } from "../shopify.server";

export class ShopifyUserError extends Error {
  constructor(public errors: { field?: string[]; message: string; code?: string }[]) {
    super(errors.map((e) => e.message).join("; "));
  }
}

/** For background jobs (no request). In routes use `authenticate.admin(request)` instead. */
export async function adminForShop(shopDomain: string) {
  const { admin } = await unauthenticated.admin(shopDomain);
  return admin;
}

export async function gql<T>(
  admin: { graphql: (q: string, o?: { variables?: unknown }) => Promise<Response> },
  query: string,
  variables?: Record<string, unknown>,
  attempt = 1,
): Promise<T> {
  const res = await admin.graphql(query, { variables });
  const body = await res.json();
  const throttled = body.errors?.some((e: any) => e.extensions?.code === "THROTTLED");
  if (throttled && attempt <= 5) {
    await new Promise((r) => setTimeout(r, 500 * 2 ** attempt));
    return gql<T>(admin, query, variables, attempt + 1);
  }
  if (body.errors?.length) throw new Error(JSON.stringify(body.errors));
  return body.data as T;
}

/** Throws if a mutation payload contains userErrors. */
export function assertNoUserErrors(payload: { userErrors?: any[] } | null | undefined) {
  if (payload?.userErrors?.length) throw new ShopifyUserError(payload.userErrors);
}
```

**0.3 Queue (pg-boss)**

```ts
// app/services/queue.server.ts
import PgBoss from "pg-boss";

let boss: PgBoss | null = null;

export async function getQueue() {
  if (!boss) {
    boss = new PgBoss({ connectionString: process.env.DATABASE_URL!, schema: "pgboss" });
    boss.on("error", (e) => console.error("[queue]", e));
    await boss.start();
  }
  return boss;
}

export async function enqueue<T extends object>(
  name: string,
  data: T,
  opts: PgBoss.SendOptions = {},
) {
  const q = await getQueue();
  await q.createQueue(name).catch(() => {}); // idempotent
  return q.send(name, data, { retryLimit: 3, retryBackoff: true, ...opts });
}
```

```ts
// app/worker.ts
import { getQueue } from "./services/queue.server";
import { registerJobs } from "./jobs";

const q = await getQueue();
await registerJobs(q); // each step adds its handlers in app/jobs/index.ts
console.log("[worker] started");
```

⚠ Verify: pg-boss v10 handler signature is `work(name, async (jobs) => …)` (array). Pin the version in `package.json`.

**0.4 Money helpers** (never use floats)

```ts
// app/services/money.server.ts
import Decimal from "decimal.js";
Decimal.set({ rounding: Decimal.ROUND_HALF_UP });

export const money = (v: string | number | Decimal) => new Decimal(v);
export const toFixed2 = (d: Decimal) => d.toDecimalPlaces(2).toFixed(2);
export const parseMoneyV2 = (m: { amount: string }) => money(m.amount);
export const percentOf = (total: Decimal, pct: Decimal) => total.mul(pct).div(100).toDecimalPlaces(2);
export function formatMoney(d: Decimal, currency: string, locale = "en-US") {
  return new Intl.NumberFormat(locale, { style: "currency", currency }).format(d.toNumber());
}
```

**0.5 Shop record on install** — call from the template's `afterAuth` hook in `shopify.server.ts`.

```ts
// app/services/shop.server.ts
const SHOP_QUERY = `#graphql
  { shop { id name email currencyCode ianaTimezone
           plan { displayName partnerDevelopment shopifyPlus } } }`;

export async function upsertShop(admin: any, shopDomain: string) {
  const { shop } = await gql<any>(admin, SHOP_QUERY);
  return prisma.shop.upsert({
    where: { shopDomain },
    create: { shopDomain, shopGid: shop.id, name: shop.name, email: shop.email,
      currencyCode: shop.currencyCode, ianaTimezone: shop.ianaTimezone,
      planName: shop.plan.displayName, isPlus: shop.plan.shopifyPlus,
      isDevStore: shop.plan.partnerDevelopment },
    update: { name: shop.name, email: shop.email, planName: shop.plan.displayName,
      isPlus: shop.plan.shopifyPlus, uninstalledAt: null },
  });
}
```

**0.6 Webhook guard** (duplicate protection)

```ts
// app/services/webhook-guard.server.ts
export async function firstDelivery(shopId: string, webhookId: string, topic: string) {
  try {
    await prisma.webhookEvent.create({ data: { shopId, webhookId, topic } });
    return true;
  } catch (e: any) {
    if (e.code === "P2002") return false; // unique violation → already received
    throw e;
  }
}
```

Every webhook route: `authenticate.webhook(request)` (HMAC check by the template) → `firstDelivery()` → **enqueue a job** → return `200` immediately (< 1 s).

**0.7 Dev tools page** `app.dev.tsx` (only when `DEV_TOOLS=true`): buttons "Run reminders now", "Re-sync order (GID)", "Run overdue scan", list of recent jobs.

### Tests
| Test | File |
|---|---|
| `percentOf(10000.01, 30)` → `3000.00`; balance `7000.01` | `tests/unit/money.test.ts` |
| Duplicate `webhookId` → `firstDelivery` returns false | `tests/integration/webhook-guard.test.ts` |
| Job that throws is retried 3 times | `tests/integration/queue.test.ts` |
| `gql` retries on `THROTTLED` | `tests/unit/shopify-admin.test.ts` (msw mock) |

### Manual check
Install on `stagepay-dev-basic` → `Shop` row exists with correct plan; trigger a test webhook (`shopify app webhook trigger --topic orders/create`) → `WebhookEvent` row, job runs in worker log.

### Done
- [ ] Shop saved on install with plan info
- [ ] Worker runs; jobs retry
- [ ] Webhook routes are fast and duplicate-safe
- [ ] Unit/integration tests green

---

## Step 1 — Deposit rules (P3, P4) (20 h)

**Goal:** merchants create rules; each active rule becomes a Shopify selling plan on the chosen products.
**Spec:** Module 1 (FR-1…FR-6). **Builds:** P3 Rules list, P4 Create/edit rule.

### Tasks

**1.1 Database** — add `DepositRule`, `RuleTarget` and enums from spec Appendix A.

```bash
npx prisma migrate dev --name step1_rules
```

**1.2 Validation schema**

```ts
// app/services/rules.server.ts
import { z } from "zod";

export const RuleInput = z.object({
  name: z.string().trim().min(1).max(60),
  depositType: z.enum(["PERCENTAGE", "FIXED"]),
  depositValue: z.coerce.number().positive(),
  minOrderValue: z.coerce.number().min(0).optional().nullable(),
  balanceTrigger: z.enum(["MANUAL", "ON_FULFILLMENT", "DAYS_AFTER_ORDER"]),
  balanceTriggerDays: z.coerce.number().int().min(1).max(365).optional().nullable(),
  finalDueDays: z.coerce.number().int().min(1).max(365).default(90),
  collectionMethod: z.enum(["SAVED_CARD", "PAYMENT_LINK"]).default("SAVED_CARD"),
  overdueDays: z.coerce.number().int().min(1).max(60).default(7),
  buyerLabel: z.string().trim().min(1).max(80),
  policyText: z.string().max(500).optional().nullable(),
  targets: z.array(z.object({
    targetType: z.enum(["PRODUCT", "COLLECTION"]),
    targetGid: z.string(), title: z.string(),
  })).min(1),
}).superRefine((r, ctx) => {
  if (r.depositType === "PERCENTAGE" && (r.depositValue < 1 || r.depositValue > 99))
    ctx.addIssue({ code: "custom", path: ["depositValue"], message: "Use 1–99%" });
  if (r.balanceTrigger === "DAYS_AFTER_ORDER" && !r.balanceTriggerDays)
    ctx.addIssue({ code: "custom", path: ["balanceTriggerDays"], message: "Enter the number of days" });
});
```

Overlap check: expand collection targets to product GIDs (`collection(id){ products(first:250, after:$cursor){ nodes{ id } pageInfo{ hasNextPage endCursor } } }`), compare with other **active** rules → error listing the conflicting products.

**1.3 Selling plan sync** (job `rules.sync`)

```ts
// app/services/selling-plans.server.ts
const CREATE = `#graphql
mutation Create($input: SellingPlanGroupInput!, $resources: SellingPlanGroupResourceInput) {
  sellingPlanGroupCreate(input: $input, resources: $resources) {
    sellingPlanGroup { id sellingPlans(first: 1) { nodes { id } } }
    userErrors { field message code }
  }
}`;

export function planInput(rule: DepositRule) {
  const checkoutCharge = rule.depositType === "PERCENTAGE"
    ? { type: "PERCENTAGE", value: { percentage: Number(rule.depositValue) } }
    : { type: "PRICE", value: { fixedValue: rule.depositValue.toFixed(2) } };
  const optionText = rule.depositType === "PERCENTAGE"
    ? `${Number(rule.depositValue)}% deposit`
    : `${rule.depositValue.toFixed(2)} deposit per item`;
  return {
    name: rule.buyerLabel,
    merchantCode: `stagepay-${rule.id}`,
    options: ["Deposit"],
    sellingPlansToCreate: [{
      name: rule.buyerLabel,
      options: optionText,
      category: "PRE_ORDER",
      billingPolicy: { fixed: {
        checkoutCharge,
        remainingBalanceChargeTrigger: "TIME_AFTER_CHECKOUT",
        remainingBalanceChargeTimeAfterCheckout: `P${rule.finalDueDays}D`,
      }},
      deliveryPolicy: { fixed: { fulfillmentTrigger: "UNKNOWN" } },
      inventoryPolicy: { reserve: "ON_SALE" },
    }],
  };
}
```

`syncRule(ruleId)` logic:

| Rule state | Shopify call(s) |
|---|---|
| Active, no group yet | `sellingPlanGroupCreate` (input + `resources.productIds`) → save `sellingPlanGroupGid`, `sellingPlanGid` |
| Active, group exists | `sellingPlanGroupUpdate` (`sellingPlansToUpdate` with the plan id) + `sellingPlanGroupAddProducts` / `sellingPlanGroupRemoveProducts` for the product diff |
| Off or deleted | `sellingPlanGroupDelete` → clear GIDs |

Set `syncStatus = SYNCED` or `FAILED` + `syncError`. After every sync, rewrite the **minimums metafield** (Step 11) and **policy metafield** (Step 2).

✅ Feasibility T1 proved `sellingPlanGroupCreate` with exactly this input works.

**1.4 Daily collection re-sync** — job `rules.resyncCollections` (cron, daily): for rules with collection targets, add newly added products.

**1.5 P3 Rules list** (`app.rules._index.tsx`)
- Loader: rules (not deleted) with targets.
- Table: Name · Deposit · Applies to · Balance due · Status toggle · Sync badge (Synced / Syncing… / Sync failed + Retry) · ⋯ Edit/Delete.
- Empty state with **Create your first rule**.
- Action intents: `toggle`, `retry-sync` → update + `enqueue("rules.sync")`.

**1.6 P4 Create/edit rule** (`app.rules.$id.tsx`, `$id = "new"` for create)
- Fields per spec Module 1 design, including **How to collect the balance** (saved card / payment link — card disabled with a tooltip if the plan can't charge saved cards, see Step 7) and **Days before overdue** (default 7); resource picker:

```ts
const selected = await shopify.resourcePicker({ type: "product", multiple: true });
// and type: "collection"
```

- `DepositPreview` component: pure function `(type, value, sampleTotal = 10000) → { today, later }`. Show "per item" for fixed deposits.
- Save action: `RuleInput.safeParse` → errors back to the form; on success upsert rule + targets in a transaction → `enqueue("rules.sync", { ruleId })` → redirect to P3.
- Delete: confirm modal "Existing orders keep their schedule" → soft delete → sync.

### Tests
| Test | Type |
|---|---|
| `RuleInput`: 0%, 100%, missing targets, DAYS without days → errors | Unit |
| `planInput` for percentage and fixed rules matches snapshot | Unit |
| Preview: 30% of $10,000 → $3,000 / $7,000 | Unit |
| Overlap: product in collection of another active rule → error | Integration |
| Save → job → `sellingPlanGroupCreate` called with expected variables (msw) | Integration |
| Toggle off → `sellingPlanGroupDelete`; edit targets → add/remove diff | Integration |
| `collectionMethod` / `overdueDays` saved and validated (1–60) | Unit |
| Shopify `userErrors` → `FAILED` + message shown | Integration |

### Manual check
Create "Custom tables — 30%" on Oak Dining Table → P3 shows *Synced* → `/products/oak-dining-table.js` shows `selling_plan_groups` with your plan.

### Done
- [ ] Create, edit, toggle, delete rules
- [ ] Each active rule synced to Shopify; failures visible and retryable
- [ ] Overlaps blocked

---

## Step 2 — Theme app extension (E1) (13 h)

**Goal:** buyers choose "Pay in full" or "Pay X% deposit" on the product page, see the amounts and policy, and add to cart with the deposit plan.
**Spec:** Module 3 (FR-7, FR-8). **Builds:** E1.

> Feasibility T10: Dawn exposes the plan in `product.selling_plan_groups` but **shows no picker**. E1 must render its own selector.

### Tasks

**2.1 Generate the extension**

```bash
shopify app generate extension --template theme_app_extension --name stagepay-block
```

**2.2 Policy metafield** (written by the app, read by Liquid)

```ts
// app/services/metafields.server.ts
const APP_INSTALLATION = `#graphql { currentAppInstallation { id } }`;
const SET = `#graphql
mutation Set($m: [MetafieldsSetInput!]!) {
  metafieldsSet(metafields: $m) { metafields { id } userErrors { field message } }
}`;

export async function setAppMetafield(admin: any, key: string, value: string, type = "multi_line_text_field") {
  const { currentAppInstallation } = await gql<any>(admin, APP_INSTALLATION);
  const data = await gql<any>(admin, SET, { m: [{
    ownerId: currentAppInstallation.id, namespace: "stagepay", key, type, value,
  }]});
  assertNoUserErrors(data.metafieldsSet);
}
```

Call `setAppMetafield(admin, "policy", text)` after rule save (Step 1) and settings save (Step 7). Per-rule policies: store JSON `{ "<sellingPlanId>": "policy text" }` under key `policies` (type `json`) and fall back to `policy`.

**2.3 Block Liquid** — `extensions/stagepay-block/blocks/stagepay-deposit.liquid`

```liquid
{%- assign group = nil -%}
{%- for g in product.selling_plan_groups -%}
  {%- if g.app_id == 'stagepay' or g.name != blank -%}{%- assign group = g -%}{%- break -%}{%- endif -%}
{%- endfor -%}

{%- if group -%}
  {%- assign plan = group.selling_plans.first -%}
  {%- assign variant = product.selected_or_first_available_variant -%}
  <div class="stagepay" data-stagepay
       data-plan-id="{{ plan.id }}"
       data-charge-type="{{ plan.checkout_charge.value_type }}"
       data-charge-value="{{ plan.checkout_charge.value }}"
       data-price="{{ variant.price }}"
       data-money-format="{{ shop.money_format | escape }}"
       data-default="{{ block.settings.default_choice }}"
       data-hide-if-native="{{ block.settings.hide_if_native }}">
    <fieldset class="stagepay__options">
      <label><input type="radio" name="stagepay-choice" value="full"> Pay in full – <span data-full></span></label>
      <label><input type="radio" name="stagepay-choice" value="deposit"> {{ plan.name | escape }} –
        <span data-today></span> today, <span data-later></span> later</label>
    </fieldset>
    {%- if block.settings.show_policy and app.metafields.stagepay.policy != blank -%}
      <p class="stagepay__policy">{{ app.metafields.stagepay.policy }}</p>
    {%- endif -%}
  </div>
{%- endif -%}

{% schema %}
{
  "name": "Stagepay deposit",
  "target": "section",
  "javascript": "stagepay.js",
  "stylesheet": "stagepay.css",
  "enabled_on": { "templates": ["product"] },
  "settings": [
    { "type": "select", "id": "default_choice", "label": "Selected by default",
      "options": [{ "value": "full", "label": "Pay in full" }, { "value": "deposit", "label": "Deposit" }],
      "default": "deposit" },
    { "type": "checkbox", "id": "show_policy", "label": "Show deposit policy", "default": true },
    { "type": "checkbox", "id": "hide_if_native", "label": "Hide if the theme already shows purchase options", "default": false },
    { "type": "color", "id": "text_color", "label": "Text colour" }
  ]
}
{% endschema %}
```

⚠ Verify: `selling_plan.checkout_charge.value_type` returns `"percentage"` or `"price"`, and `.value` is the percentage or the amount in cents. Identify Stagepay groups reliably by matching `selling_plan.id` against a JSON metafield of Stagepay plan IDs (`app.metafields.stagepay.plan_ids`) written on sync — the `app_id` check above is a placeholder.

**2.4 Block JavaScript** — `assets/stagepay.js`

```js
(() => {
  document.querySelectorAll("[data-stagepay]").forEach((root) => {
    const section = root.closest("section, .shopify-section") || document;
    const form = section.querySelector('form[action*="/cart/add"]');
    if (!form) return;

    if (root.dataset.hideIfNative === "true" && form.querySelector('[name="selling_plan"]:not([data-stagepay-input])')) {
      root.hidden = true; return;
    }

    let input = form.querySelector('input[name="selling_plan"][data-stagepay-input]');
    if (!input) {
      input = Object.assign(document.createElement("input"), { type: "hidden", name: "selling_plan" });
      input.dataset.stagepayInput = "";
      form.appendChild(input);
    }

    const qtyEl = () => form.querySelector('[name="quantity"]') || section.querySelector('[name="quantity"]');
    const fmt = (cents) => (window.Shopify?.formatMoney
      ? Shopify.formatMoney(cents, root.dataset.moneyFormat)
      : (cents / 100).toFixed(2));

    function render() {
      const qty = Number(qtyEl()?.value || 1);
      const price = Number(root.dataset.price);            // cents
      const total = price * qty;
      const value = Number(root.dataset.chargeValue);
      const today = root.dataset.chargeType === "percentage"
        ? Math.round((total * value) / 100)
        : Math.min(value * qty, total);
      root.querySelector("[data-full]").textContent = fmt(total);
      root.querySelector("[data-today]").textContent = fmt(today);
      root.querySelector("[data-later]").textContent = fmt(total - today);
      const choice = root.querySelector('input[name="stagepay-choice"]:checked')?.value;
      input.value = choice === "deposit" ? root.dataset.planId : "";
      input.disabled = choice !== "deposit"; // nothing sent for "Pay in full"
    }

    const def = root.querySelector(`input[value="${root.dataset.default}"]`);
    if (def) def.checked = true;
    root.addEventListener("change", render);
    form.addEventListener("change", render);       // quantity
    document.addEventListener("variant:change", (e) => {  // theme-specific event
      const v = e.detail?.variant; if (v) { root.dataset.price = v.price; render(); }
    });
    render();
  });
})();
```

⚠ Verify per theme: the variant-change event name differs (Dawn publishes via its pub/sub system). Fallback: listen to `change` on `[name="id"]` and read the price from the product JSON (`/products/<handle>.js`, fetched once).

**2.5 Styles** — `assets/stagepay.css`: inherit theme fonts, spacing 12–16 px, radio rows with clear focus outline, policy in smaller muted text.

**2.6 Theme compatibility** — test on Dawn + two other free OS 2.0 themes (e.g. Refresh, Sense). Record the variant-change behaviour for each in `tests/manual/themes.md`.

### Tests
| Test | Type |
|---|---|
| Amount calculator (extract to a pure function): 30% × 10 × $1,000 → $3,000 / $7,000; fixed $500 × 3 | Unit (Vitest with jsdom) |
| Choice "full" → hidden input disabled; "deposit" → plan id set | Unit (jsdom) |
| Product page → choose deposit → add to cart → cart line shows plan name | E2E (Playwright, dev store) |
| Checkout shows due today $3,000 / due later $7,000 | E2E |

### Manual check
`shopify app dev` → theme editor → add **Stagepay deposit** block to the product template → storefront: choose deposit, quantity 10, add to cart → cart shows "Pay 30% deposit now, rest later" (same as your feasibility screenshot) → checkout shows $3,000 today.

### Done
- [ ] Selector works on Dawn + 2 themes; amounts update with quantity and variant
- [ ] "Pay in full" adds without a plan; "Deposit" adds with the plan
- [ ] Policy shown
- [ ] 🎯 **Milestone: buyers can pay a deposit**

---

## Step 3 — Order sync & status engine (20 h)

**Goal:** every deposit order is saved in Stagepay with correct amounts and statuses, kept in sync with Shopify.
**Spec:** Module 2. **Builds:** background only.

### Tasks

**3.1 Database** — add `DepositOrder`, `PaymentStep` and enums (Appendix A).

```bash
npx prisma migrate dev --name step3_orders
```

**3.2 Webhook route** — one route for all order topics.

```ts
// app/routes/webhooks.orders.tsx
export const action = async ({ request }: ActionFunctionArgs) => {
  const { shop, topic, payload, webhookId } = await authenticate.webhook(request);
  const s = await prisma.shop.findUnique({ where: { shopDomain: shop } });
  if (!s || !(await firstDelivery(s.id, webhookId, topic))) return new Response();

  const orderGid = topic.startsWith("FULFILLMENTS")
    ? `gid://shopify/Order/${payload.order_id}`
    : topic.startsWith("ORDER_TRANSACTIONS") || topic.startsWith("REFUNDS")
      ? `gid://shopify/Order/${payload.order_id}`
      : payload.admin_graphql_api_id;

  await enqueue("orders.sync", { shopId: s.id, orderGid, topic },
    { singletonKey: `${s.id}:${orderGid}`, singletonSeconds: 5 }); // collapse bursts
  return new Response();
};
```

⚠ Verify: topic string format returned by `authenticate.webhook` (e.g. `ORDERS_CREATE`) and payload field names, using the saved fixtures from feasibility test T9.

**3.3 Order query** — reuse the T2 query (now with `read_payment_terms`):

```graphql
query StagepayOrder($id: ID!) {
  order(id: $id) {
    id name createdAt cancelledAt displayFinancialStatus tags note
    customer { id displayName email }
    currentTotalPriceSet { shopMoney { amount currencyCode } }
    totalReceivedSet     { shopMoney { amount } }
    totalRefundedSet     { shopMoney { amount } }
    totalOutstandingSet  { shopMoney { amount } }
    paymentTerms { paymentSchedules(first: 5) { nodes { id dueAt completedAt } } }
    paymentCollectionDetails { additionalPaymentCollectionUrl vaultedPaymentMethods { id } }
    fulfillments(first: 5) { createdAt }
    lineItems(first: 100) { nodes { sellingPlan { sellingPlanId name } } }
    transactions(first: 50) { id kind status processedAt gateway amountSet { shopMoney { amount } } }
  }
}
```

✅ Feasibility T2 confirmed: `PARTIALLY_PAID`, received/outstanding amounts, `sellingPlanId`, mandate and payment URL are all returned.

**3.4 Pure status function** (unit-test this heavily)

```ts
// app/services/order-status.server.ts
export function computeStatus(o: {
  cancelled: boolean; outstanding: Decimal; balanceStep: { status: PaymentStepStatus };
  balanceDueAt: Date | null; now: Date;
}): DepositOrderStatus {
  if (o.cancelled) return "CANCELLED";
  if (o.outstanding.lte(0)) return "PAID";
  if (o.balanceStep.status === "FAILED") return "PAYMENT_FAILED";
  if (o.balanceStep.status === "REQUESTED" && o.balanceDueAt && o.now > o.balanceDueAt) return "OVERDUE";
  if (["REQUESTED", "PROCESSING"].includes(o.balanceStep.status)) return "BALANCE_REQUESTED";
  return "DEPOSIT_PAID";
}
```

**3.5 `syncOrder(shopId, orderGid)`** (job `orders.sync`)
1. Load order (3.3). If no line item has a Stagepay `sellingPlanId` → ignore.
2. **First time:** create `DepositOrder` (copy `collectionMethod` and `overdueDays` from the rule — later rule edits must not change existing orders) + steps — Step 1 *Deposit* = received, `PAID`; Step 2 *Balance* = outstanding, `WAITING`, trigger from the rule. If trigger `DAYS_AFTER_ORDER` → `enqueue("balance.request", …, { startAfter })` (Step 5 adds the handler). Increment usage (Step 10 adds the counter).
3. **Every time:** overwrite total/paid/outstanding from Shopify; step 2 amount = outstanding while not paid; match new successful `SALE`/`CAPTURE` transactions to the lowest unpaid step; set `fulfilledAt`; compute status.
4. If topic is fulfillment and trigger is `ON_FULFILLMENT` and step 2 `WAITING` → `enqueue("balance.request", { reason: "fulfillment" })`.
5. **Write back to Shopify only when changed:**
   - Tags (`tagsAdd` / `tagsRemove`): `stagepay-deposit-paid`, `stagepay-balance-due`, `stagepay-overdue`, `stagepay-paid`.
   - Metafield `stagepay.schedule` (JSON of steps) on the order (`metafieldsSet`, owner = order).
   - Note line (`orderUpdate(input:{id, note})`): append `[Stagepay] 2026-11-03 Deposit $3,000.00 paid` — only once per event (keep a hash of written events in `DepositOrder`).

**3.6 Overdue scan** — job `orders.overdueScan`, scheduled hourly; for each shop, steps `REQUESTED` with `balanceDueAt < now` → re-run status (Step 6 adds the staff email).

### Tests
| Test | Type |
|---|---|
| `computeStatus` table: every row of the spec state machine | Unit |
| Transaction matching (deposit → step 1, later → step 2, refund ignored) | Unit |
| `orders/create` fixture (from T9) with plan → order + 2 steps; without plan → ignored | Integration |
| Same webhook twice → one sync | Integration |
| Edit 20 → 25 tables → total 12,500, owed 9,500 | Integration (fixture) |
| Cancelled → CANCELLED, scheduled jobs removed | Integration |

### Manual check
Place a deposit order through E1 → within 10 s: DB row with 10,000 / 3,000 / 7,000; in Shopify admin the order has tags `stagepay-deposit-paid`, `stagepay-balance-due` and a `[Stagepay]` note line.

### Done
- [ ] Deposit orders saved within 10 s, amounts always equal Shopify's
- [ ] Edits, refunds, cancellations reflected; duplicates ignored
- [ ] Tags, metafield and note written

---

## Step 4 — Dashboard, simple (P1 part) (5 h)

**Goal:** see and open deposit orders.
**Spec:** Module 6. **Builds:** P1 (orders list only).

### Tasks
1. Loader `listOrders(shopId, { tab, q, cursor, sort })`:
   - Tabs map to statuses: All · Balance due (`DEPOSIT_PAID`) · Requested · Overdue · Payment failed · Paid · Cancelled.
   - Search `orderName`, `customerName`, `customerEmail` (`contains`, case-insensitive).
   - Cursor paging 50 per page, sort by `createdAt desc`.
2. UI: tabs with counts, search field, table: Order · Customer · Total · Paid · Owed · Status badge · Created. Row click → `/app/orders/:id`.
3. `StatusBadge` component (tone per status, spec Module 6).
4. Empty state: "No deposit orders yet" + link to P3.

### Tests
| Test | Type |
|---|---|
| Tab filter and search return correct rows (seeded DB) | Integration |
| Shop A cannot see Shop B's orders | Integration |

### Done
- [ ] Orders listed with correct amounts and statuses; tabs, search, paging work

---

## Step 5 — Order detail & balance collection (P2) (28 h)

**Goal:** staff collect the balance by saved card or payment link, or record a bank transfer — always on the same order, never twice.
**Spec:** Module 4 + Module 6 (order detail). **Builds:** P2 with pop-ups.

### Tasks

**5.1 Database** — add `PaymentAttempt`, `AttemptStatus` (Appendix A).

```bash
npx prisma migrate dev --name step5_attempts
```

**5.2 P2 page** (`app.orders.$id.tsx`)
- Header: order name (link `shopify:admin/orders/<id>`), customer, status badge.
- Critical banner when outstanding > 0: **"Do not ship: $7,000.00 unpaid"**.
- Money card: Total · Paid · Owed · "Last synced …" + **Refresh** (intent `refresh` → `syncOrder`).
- Steps timeline with attempts history.
- Actions: **Collect balance** · **Send payment link again** · **Record payment** · **Cancel balance** (visibility rules in spec Module 4).

**5.3 `requestBalance(depositOrderId, method, actor)`** (job `balance.request`)

```ts
// app/services/balance.server.ts
export async function requestBalance(id: string, method: CollectionMethod, actor?: string) {
  const step = await prisma.$transaction(async (tx) => {
    const [s] = await tx.$queryRaw<PaymentStep[]>`
      SELECT * FROM "PaymentStep" WHERE "depositOrderId" = ${id} AND position = 2 FOR UPDATE`;
    if (!["WAITING", "FAILED"].includes(s.status) && !(await isOverdue(tx, id))) return null; // double click → no-op
    return tx.paymentStep.update({ where: { id: s.id }, data: { status: "PROCESSING", method } });
  });
  if (!step) return;

  const order = await syncOrder(/* … */);            // fresh amounts from Shopify
  if (order.outstandingAmount.lte(0)) return;         // already paid

  const useCard = method === "SAVED_CARD" && order.mandateGid && features(order.shopId).savedCardCharge;
  if (useCard) await enqueue("balance.charge", { stepId: step.id });
  else await sendPaymentLink(step.id);

  await markRequested(step.id);    // REQUESTED, requestedAt, balanceDueAt = now + order.overdueDays
  // Step 6: scheduleReminders(step.id)
  // Step 12: audit("balance.requested", …)
}
```

**5.4 Charge saved card** (jobs `balance.charge`, `balance.pollCharge`)

```graphql
mutation StagepayCharge($id: ID!, $mandateId: ID!, $key: String!) {
  orderCreateMandatePayment(id: $id, mandateId: $mandateId, idempotencyKey: $key, autoCapture: true) {
    job { id done }
    paymentReferenceId
    userErrors { field message code }
  }
}
```

```graphql
query StagepayPaymentStatus($orderId: ID!, $ref: String!) {
  orderPaymentStatus(orderId: $orderId, paymentReferenceId: $ref) { status errorMessage }
}
```

1. Create `PaymentAttempt` (`PENDING`, `idempotencyKey = stagepay:<stepId>:<n>`) **before** calling Shopify. Retries of the same attempt reuse the key.
2. Call the mutation **without `amount`** (whole balance in v1).
3. `enqueue("balance.pollCharge", …, { startAfter: 15 })` — repeat up to 10 minutes:
   - success status → attempt `SUCCEEDED` → `syncOrder` (order becomes `PAID`).
   - error status → attempt `FAILED` → `handleFailure` (5.7).
   - still pending after 10 min → leave `PENDING`, log; the transaction webhook will resolve it.

⚠ Verify the exact success/failure values of `orderPaymentStatus.status` with feasibility T4 and T8 results.

**5.5 Payment link**
- URL = `order.paymentCollectionDetails.additionalPaymentCollectionUrl` (✅ present in T2).
- Save `paymentLinkUrl`; in Step 5 just show it on P2 with **Copy link**; Step 6 adds the email.
- Fallback if T6 shows the link creates a new order: use Shopify's `orderInvoiceSend`.

**5.6 Record payment**

```graphql
mutation StagepayManual($id: ID!, $amount: MoneyInput!, $name: String, $at: DateTime) {
  orderCreateManualPayment(id: $id, amount: $amount, paymentMethodName: $name, processedAt: $at) {
    order { id displayFinancialStatus totalOutstandingSet { shopMoney { amount } } }
    userErrors { field message }
  }
}
```

Validate `0 < amount ≤ outstanding`; attempt `MANUAL_RECORD` → `SUCCEEDED` → `syncOrder`. Part payments leave the step `REQUESTED`.

**5.7 Failure handling** — step `FAILED`, order `PAYMENT_FAILED`; Step 6 adds buyer/staff emails. P2 shows the error message and enables **Collect balance** again.

**5.8 Triggers** — register `balance.request` handler for:
- `MANUAL` → P2 button,
- `ON_FULFILLMENT` → enqueued by Step 3,
- `DAYS_AFTER_ORDER` → delayed job from Step 3.
For every trigger the method comes from the order (`order.collectionMethod`, copied from its rule). The P2 pop-up pre-selects it; staff may switch.

**5.9 Pop-ups** — `CollectBalanceModal` (method radio: saved card / payment link; amount read-only), `RecordPaymentModal` (amount, method select, date, reference). Toasts: "Charge started", "Balance paid", "Charge failed".

### Tests
| Test | Type |
|---|---|
| Request on PAID / CANCELLED / PROCESSING step → no-op | Unit |
| Manual amount 0, negative, > outstanding → errors | Unit |
| Two concurrent `requestBalance` → exactly one `orderCreateMandatePayment` | Integration |
| Retry after network error reuses the idempotency key | Integration |
| Poll success → PAID; poll error → FAILED | Integration (msw) |
| Manual 4,000 then 3,000 → PAID after second | Integration |
| Fulfillment trigger fires once for two fulfillments | Integration |
| Card success / decline / link payment / bank transfer on dev store | Manual (feasibility T4, T6, T7, T8 steps) |

### Manual check
On `stagepay-dev-basic`: new deposit order → P2 → **Collect balance (card)** → order *Paid* in Shopify, one order only. Repeat with link (pay in private window) and with **Record payment**.

### Done
- [ ] Card, link and manual payments all land on the same order
- [ ] No duplicate charges (double click, retries, duplicate webhooks)
- [ ] Failure visible on P2
- [ ] 🎯 **Milestone: works end to end — demo to a merchant**

---

## Step 6 — Emails & reminders (13 h)

**Goal:** buyers get balance, receipt, reminder and failure emails; staff get alerts; nobody chases by hand.
**Spec:** Module 5. **Builds:** background (templates use defaults; editor comes in Step 8).

### Tasks

**6.1 Database** — add `EmailTemplate`, `EmailLog`, `Reminder`, `EmailKey`, `EmailStatus`.

```bash
npx prisma migrate dev --name step6_emails
```

**6.2 Provider**

```ts
// app/services/email-provider.server.ts
import { ServerClient } from "postmark";
const client = new ServerClient(process.env.POSTMARK_TOKEN!);

export async function sendRaw(m: { from: string; to: string; replyTo?: string;
  subject: string; html: string; text: string; tag: string }) {
  if (process.env.EMAIL_TEST_MODE === "true") { console.log("[email:test]", m.to, m.subject); /* or Mailtrap SMTP */ }
  const r = await client.sendEmail({ From: m.from, To: m.to, ReplyTo: m.replyTo,
    Subject: m.subject, HtmlBody: m.html, TextBody: m.text, Tag: m.tag, MessageStream: "outbound" });
  return r.MessageID;
}
```

**6.3 Defaults** — `email-defaults.server.ts` holds subject + body for the 7 keys (spec Module 5 table). Placeholders: `{{shop_name}} {{customer_name}} {{order_name}} {{order_total}} {{paid}} {{balance}} {{pay_link}} {{due_date}} {{policy}}`.

**6.4 Render & send** (`email.server.ts`)
- `render(template, vars)`: replace placeholders; **HTML-escape every value**; `{{pay_link}}` becomes a button.
- Layout: logo (Step 7), body, Pay button, footer with shop name.
- `sendEmail(shopId, key, depositOrderId)`: template (DB or default) → `EmailLog QUEUED` → send → `SENT` / `FAILED` (job retries 3×).
- From: `"<Shop name>" <notifications@${EMAIL_FROM_DOMAIN}>`; Reply-To: shop email (settings in Step 7).

**6.5 Wire emails into earlier steps**
| Event (step) | Email |
|---|---|
| Link requested (5.5) | `BALANCE_DUE` |
| Card charge succeeded (5.4) | `BALANCE_CHARGED` |
| Outstanding reaches 0 by link/manual (3.5) | `FULLY_PAID` |
| Card failed (5.7) | `CARD_FAILED` (buyer) + `STAFF_CARD_FAILED` |
| Order becomes OVERDUE (3.6) | `STAFF_OVERDUE` |

**6.6 Reminders** (`reminders.server.ts`)
- `scheduleReminders(stepId)`: return early if `settings.remindersEnabled` is false; for each day in `settings.reminderDays` (default `[3, 7, 14]`, edited on P1 in Step 9) create `Reminder` + `enqueue("reminders.send", { reminderId }, { startAfter: seconds(day) })`. With `TIME_COMPRESSION=1440`, 1 day = 60 s.
- Job `reminders.send`: skip if step `PAID`/`CANCELLED` or reminder cancelled → send `REMINDER`.
- `cancelReminders(stepId)` when paid, cancelled or balance cancelled.

**6.7 Delivery status** — `webhooks.postmark.tsx` (basic-auth secret) → update `EmailLog` to `DELIVERED` / `BOUNCED`.

**6.8 P2** — add an **Emails** card (type, to, status, date).

### Tests
| Test | Type |
|---|---|
| Placeholder rendering; `<script>` in customer name is escaped | Unit |
| Reminder times for [3,7,14] from requestedAt (and with compression) | Unit |
| Step PAID → pending reminders cancelled, nothing sent | Integration |
| Provider error → `FAILED`, retried | Integration |
| All 7 emails render in Gmail/Outlook/mobile (Mailtrap) | Manual |

### Done
- [ ] All 7 emails send with correct amounts and links
- [ ] Reminders stop immediately on payment or cancel
- [ ] Every email visible on P2
- [ ] 🎯 **Milestone: no manual chasing**

---

## Step 7 — Settings (P5) (3 h)

**Goal:** merchants change branding, email addresses and default policy; features follow the Shopify plan. (Reminders are on P1 — Step 9; collection method and days before overdue are per rule — Step 1.)
**Spec:** Module 7. **Builds:** P5.

### Tasks
1. **Database** — add `ShopSettings`; create defaults on install (`replyToEmail = shop.email`).
   ```bash
   npx prisma migrate dev --name step7_settings
   ```
2. **Validation** (zod): emails valid.
3. **P5 page** sections: Branding (logo upload, colour) · Emails (reply-to, staff alert emails, link to P6) · Policy (default text).
4. **Logo upload** — `stagedUploadsCreate` → upload → `fileCreate` → store the file URL.
5. **On save** — update settings; `setAppMetafield("policy", …)` (E1 reads it).
6. **Replace hard-coded values** from Step 6: reply-to, staff alert emails, logo.
7. **Plan detection** (`plan.server.ts`) — on install and `shop/update` webhook refresh `planName/isPlus`; compute `features` from plan + feasibility results:
   ```ts
   export function featuresFor(shop: Shop) {
     return {
       savedCardCharge: true,            // set from feasibility T4 per plan
       partialCharge: shop.isPlus,       // from T5; used in later phases
     };
   }
   ```
   P2 and P5 disable unavailable options with a tooltip ("Not available on your Shopify plan — payment link is used instead").

### Tests
| Test | Type |
|---|---|
| Validation errors (bad email) | Unit |
| Install → defaults created | Integration |
| `shop/update` Grow → Plus → features updated | Integration |

### Done
- [ ] Settings persist and change email sender, alerts and branding
- [ ] Plan-gated features hidden or explained

---

## Step 8 — Email templates editor (P6) (3 h)

**Goal:** merchants edit the wording of the 7 emails.
**Spec:** Module 5 (editor). **Builds:** P6.

### Tasks
1. **List** (`app.settings.emails._index.tsx`): 7 emails, enabled toggle, "Edited" badge if not default.
2. **Editor** (`app.settings.emails.$key.tsx`): subject + body (bold, links, line breaks only — sanitize on save), **Insert placeholder** menu, live preview with a sample order, **Send test** (to staff email), **Reset to default**.
3. Save into `EmailTemplate` (upsert by `shopId + key`); disabled templates are skipped and logged as skipped.

### Tests
| Test | Type |
|---|---|
| Sanitizer removes scripts/styles, keeps bold and links | Unit |
| Disabled template → not sent | Integration |
| Reset restores default | Integration |

### Done
- [ ] All 7 templates editable, previewable, testable, resettable

---

## Step 9 — Dashboard, full (P1 finish) (7 h)

**Goal:** totals at a glance, reminder controls, and a setup checklist for new merchants.
**Spec:** Module 6 + 7 (onboarding). **Builds:** P1 completed.

### Tasks
1. **Summary query** (single SQL, < 300 ms at 5,000 orders):
   ```sql
   SELECT
     SUM("paidAmount")        FILTER (WHERE status NOT IN ('PAID','CANCELLED')) AS deposits_held,
     SUM("outstandingAmount") FILTER (WHERE status NOT IN ('PAID','CANCELLED')) AS balances_owed,
     SUM("outstandingAmount") FILTER (WHERE status = 'OVERDUE')                 AS overdue
   FROM "DepositOrder" WHERE "shopId" = $1 AND untracked = false;
   ```
   "Paid this month" = sum of step 2 attempts `SUCCEEDED` with `completedAt` in the current month (shop timezone).
2. **Cards:** Deposits held · Balances owed · Overdue · Paid this month.
3. **Indexes** for search: `pg_trgm` GIN indexes on `orderName`, `customerName`, `customerEmail` (raw SQL migration).
4. **Reminders panel** on P1: "Automatic reminders" On/Off switch + reminder days as chips (default 3, 7, 14) + **Save**.
   - Action `intent=save-reminders`: validate (zod: days 1–60, ascending, max 5) → save `remindersEnabled`, `reminderDays` in `ShopSettings`.
   - Turned **off** → `cancelReminders` for every pending reminder of the shop.
   - Turned **on** or days changed → for steps in `REQUESTED`, cancel old pending reminders and `scheduleReminders` again (only future dates).
   - Help text: "Sent to buyers after a balance is requested, until it's paid."
5. **Setup checklist** (stored in `ShopSettings.onboardingDone`): ① Create a rule ② Add block — deep link
   `https://{shop}/admin/themes/current/editor?template=product&addAppBlockId={SHOPIFY_API_KEY}/stagepay-deposit&target=mainSection`
   ③ Place a test order (shows test card numbers) ④ Collect a test balance. Auto-tick ① and ③/④ from data; ② when the merchant clicks. Dismissible.

### Tests
| Test | Type |
|---|---|
| Summary numbers on seeded mixed data | Integration |
| Reminders off → pending reminders cancelled; on → rescheduled; bad days rejected | Integration |
| 5,000 seeded orders → loader < 300 ms | Perf (CI) |

### Done
- [ ] Totals match Shopify orders on the dev store
- [ ] Reminders panel turns reminders on/off and changes days
- [ ] Checklist guides a new merchant to the first collected balance

---

## Step 10 — Plan & billing (P7) (5 h)

**Goal:** Free (10 deposit orders/month) and Pro ($29/month), billed by Shopify.
**Spec:** Module 8 (FR-21). **Builds:** P7.

### Tasks
1. **Database** — add `BillingState`, `AppPlan`; `DepositOrder.untracked` already exists.
2. **Template billing config** (`shopify.server.ts`):
   ```ts
   export const PRO_PLAN = "Stagepay Pro";
   // in shopifyApp({...})
   billing: {
     [PRO_PLAN]: { lineItems: [{ amount: 29, currencyCode: "USD", interval: BillingInterval.Every30Days }] },
   },
   ```
3. **P7 page:** Free vs Pro cards, current plan, usage bar ("7 of 10 this month"), **Upgrade** → `billing.request({ plan: PRO_PLAN, isTest: shop.isDevStore, returnUrl })`, **Downgrade** → `billing.cancel(...)`.
4. **Webhook** `app_subscriptions/update` → set plan/status. On `ACTIVE`: set `untracked = false` for this period's orders, re-sync them, schedule due triggers/reminders.
5. **Usage:** increment `ordersThisPeriod` when Step 3 creates an order; on Free with ≥ 10 → create the order with `untracked = true` (no reminders, actions disabled, greyed in P1 with upgrade prompt).
6. **Reset job** `billing.resetUsage` — 1st of each month (UTC).
7. **P1 banner** at 8/10 and 10/10.

### Tests
| Test | Type |
|---|---|
| Limit at exactly 10 / 11 | Unit |
| Upgrade webhook → PRO; untracked orders tracked and re-synced | Integration |
| 11th order on Free → untracked, no reminder jobs | Integration |
| Approve and decline test charge on dev store | Manual |

### Done
- [ ] Free limit enforced; Pro unlocks everything; buyers never blocked
- [ ] 🎯 **Milestone: ready to sell**

---

## Step 11 — Checkout validation Function (E2) (4 h)

**Goal:** block checkout when a deposit order is below the rule's minimum order value.
**Spec:** Module 3 (FR-3). **Builds:** E2.

### Tasks
1. **Generate**
   ```bash
   shopify app generate extension --template cart_checkout_validation --name stagepay-min-order --flavor typescript
   ```
2. **Minimums metafield** — on every rule sync write `stagepay.minimums` (JSON `{ "<sellingPlanId>": "5000.00" }`) where the Function can read it. ⚠ Verify which owner the Function input can read (validation-owned metafield set via `validationCreate(metafields: …)` is the documented pattern).
3. **Activate** — on first rule with a minimum, call `validationCreate` (title "Stagepay minimum order", enable, block on failure); keep its id in settings.
4. **Input query** (`src/run.graphql`, adjust to the generated schema):
   ```graphql
   query RunInput {
     cart {
       cost { subtotalAmount { amount } }
       lines { sellingPlanAllocation { sellingPlan { id } } }
     }
     validation { metafield(namespace: "$app:stagepay", key: "minimums") { jsonValue } }
   }
   ```
   ⚠ Verify `sellingPlanAllocation` is available on Function cart lines. Fallback: E1 adds a hidden line property `_stagepay_plan` and the Function reads `attribute(key: "_stagepay_plan")`.
5. **Logic** (`src/run.ts`): for each line with a Stagepay plan that has a minimum, if `subtotal < minimum` → error `"Deposit option requires an order of at least $5,000. Choose Pay in full or add more items."` on target `cart`.
6. **Deploy** — `shopify app deploy` (Functions only run after deploy).

### Tests
| Test | Type |
|---|---|
| Below minimum → error; above → none; no Stagepay plan → none | Unit (Function test runner) |
| Checkout with 3 tables ($3,000) and 30% plan, minimum $5,000 → blocked | Manual |
| 10 tables → allowed | Manual |

### Done
- [ ] Checkout blocked below the minimum with a clear message; no effect otherwise

---

## Step 12 — Privacy, uninstall, activity log (7 h)

**Goal:** meet App Store requirements and keep a record of every money action.
**Spec:** Module 9. **Builds:** background + Activity card on P2.

### Tasks
1. **Database** — add `AuditLog`, `ActorType`.
2. **Audit** — `audit(shopId, action, details, actor)`; call from rules (created/updated/deleted), balance (requested, charged, failed, recorded, cancelled), settings saved, plan changed. Staff identity from the session (`session.onlineAccessInfo?.associated_user` if online tokens are used; otherwise "Staff").
3. **Activity card** on P2: "Nov 3, 10:42 — Sarah collected balance by card — $7,000.00 — Succeeded".
4. **Compliance webhook** (`webhooks.compliance.tsx`):
   | Topic | Action |
   |---|---|
   | `customers/data_request` | Collect that customer's `DepositOrder` + `EmailLog` rows → email JSON to the shop owner; audit |
   | `customers/redact` | Replace name/email with `redacted` in those rows |
   | `shop/redact` | Delete all rows for the shop (cascade) |
5. **Uninstall** (`webhooks.app.uninstalled.tsx`): set `uninstalledAt`, delete sessions, cancel the shop's queued jobs, `enqueue("shop.purge", …, { startAfter: 48h })`; on reinstall within 48 h cancel the purge. ⚠ Use the result of feasibility **T12**: if Stagepay's selling plans stay on products after uninstall, add a **"Remove all deposit options"** button in P5 (and mention it in the uninstall help text).
6. **Security pass**: every loader/action uses `authenticate.admin` and filters by shop; rate limit POST actions (30/min per shop); secrets only in env; DB encrypted at rest with daily backups.

### Tests
| Test | Type |
|---|---|
| Each compliance fixture → expected data change | Integration |
| Uninstall → purge after 48 h; reinstall cancels | Integration |
| Cross-shop access to every route → 404 | Security |
| Bad HMAC → 401 | Security |
| Collect balance → audit row with actor | Integration |

### Done
- [ ] All mandatory webhooks implemented; uninstall cleanup works
- [ ] Every money action in the activity log

---

## Step 13 — Full testing & launch (29 h)

**Goal:** prove everything works on both plans and submit to the App Store.
**Spec:** Module 10.

### 13.1 Acceptance tests (run on `stagepay-dev-basic` and `stagepay-dev-plus`)

| # | Test | Pass |
|---|---|---|
| A1 | 30% deposit on a $10,000 order via E1 | ONE order, 3,000 paid, 7,000 owed |
| A2 | Collect balance by saved card | Same order Paid, receipt email |
| A3 | Collect balance by link | Same order Paid, no new order |
| A4 | Record $7,000 bank transfer | Paid, reminders stop |
| A5 | Trigger "on shipping" | Requested automatically on fulfillment |
| A6 | Reminders 3/7/14 (compressed time) | Sent, then stop on payment |
| A7 | Declined card | Buyer link email, staff alert, Payment failed |
| A8 | Double click Collect balance | One charge |
| A9 | Edit order 20 → 25 tables | Owed updates |
| A10 | Dashboard totals | Match Shopify |
| A11 | Free plan 11th order; upgrade | Untracked → tracked after Pro |
| A12 | Uninstall + compliance webhooks | Data handled |
| A13 | Minimum order (E2) | Blocked below, allowed above |
| A14 | E1 on Dawn + 2 themes | Selector and amounts correct |

Bug fixing: log every failure as an issue, fix, re-run the affected rows.

### 13.2 Non-functional checks
| Check | Target |
|---|---|
| Admin pages | < 2 s with 5,000 orders |
| Webhook response | < 1 s |
| Product page | ≤ 2 Lighthouse points slower with E1 |
| Errors | visible in error tracker within 5 min |

### 13.3 Launch checklist
- [ ] Production DB (backups verified), web + worker processes running, error tracking and uptime alerts on
- [ ] `shopify app deploy` (extensions + Function), production app URL set
- [ ] Email domain `mail.stagepay.app` verified (SPF, DKIM, Return-Path)
- [ ] Protected scopes approved
- [ ] Billing set to live (`isTest` false for non-dev stores)
- [ ] Listing: **Stagepay: Milestone Payments**, tagline, description, 5 screenshots (P1, P2, P4, E1, email), demo video, support email, privacy policy URL, FAQ
- [ ] Setup guide + user guide published
- [ ] 2–3 beta merchants installed via direct link; one small live order ($10 product, $3 deposit) collected and refunded
- [ ] Submit for review
- [ ] 🎯 **Milestone: ready for App Store review**

---

## Appendix A — Command cheat sheet

```bash
shopify app dev                       # run app + tunnel; press g for GraphiQL
```
```bash
npm run worker                        # run job worker (second terminal)
```
```bash
npx prisma migrate dev --name <name>  # new migration
```
```bash
npx prisma studio                     # browse the database
```
```bash
shopify app webhook trigger --topic orders/create --address <url>   # test webhook
```
```bash
shopify app generate extension        # new extension
```
```bash
shopify app deploy                    # deploy extensions, Function and config
```
```bash
npx vitest run                        # unit + integration tests
```
```bash
npx playwright test                   # E2E tests against the dev store
```

Test cards: `4242 4242 4242 4242` (success), `4000 0000 0000 0002` (decline); Bogus Gateway `1` success, `2` failure.

---

## Appendix B — Definition of done (every step)

- [ ] Code merged to `main` through a reviewed pull request
- [ ] Migrations included and applied cleanly on a fresh database
- [ ] Unit and integration tests for the step written and green in CI
- [ ] Manual check done on `stagepay-dev-basic` (and `stagepay-dev-plus` for payment steps)
- [ ] No secrets in code; errors logged with shop and order context
- [ ] Spec updated if behaviour changed (and the change noted in its version line)

---

## Appendix C — Git workflow

- Branch per step: `step-<n>-<short-name>` (e.g. `step-5-balance-collection`).
- Small commits with clear messages; one pull request per step, linked to this guide's step.
- Tag milestones: `v0.2-deposit` (after Step 2), `v0.5-demo` (Step 5), `v0.10-sellable` (Step 10), `v1.0.0` (submission).
- Save webhook payloads from feasibility tests in `tests/fixtures/` and reuse them in integration tests.
