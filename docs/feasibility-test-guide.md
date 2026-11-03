# Feasibility Test Guide — before development starts

Version 1.0 · 2026-11-02 · Companion to `docs/deposit-app-spec.md` (Module 0, spike S1–S7)

**Goal:** prove, in 1–2 days and without real money, that Shopify lets an app do the four things the whole product depends on:

1. Take a **deposit at checkout** and create **one partly-paid order**.
2. **Charge the saved card** for the remaining balance.
3. Let the buyer **pay the balance by link on the same order**.
4. **Record a bank transfer** on the order.

Plus two "nice to know" answers for later phases: can the app charge **part** of the balance, and does any of this differ between **non-Plus** and **Plus** stores?

No real money is used. Everything runs on free development stores with test cards.

---

## Pass / fail summary (fill this in at the end)

| # | Test | Non-Plus store | Plus store | Needed for |
|---|---|---|---|---|
| T1 | Create a deposit option (selling plan) by API | ☐ Pass ☐ Fail | ☐ Pass ☐ Fail | v1 — **must pass** |
| T2 | Deposit checkout → ONE order, *Partially paid* | ☐ ☐ | ☐ ☐ | v1 — **must pass** |
| T3 | Saved card available on the order | ☐ ☐ | ☐ ☐ | v1 (card charge) |
| T4 | Charge the **whole** balance on the saved card | ☐ ☐ | ☐ ☐ | v1 (card charge) |
| T5 | Charge **part** of the balance on the saved card | ☐ ☐ | ☐ ☐ | Later (3+ steps) |
| T6 | Balance payment link pays the **same** order | ☐ ☐ | ☐ ☐ | v1 (link) |
| T7 | Record a manual (bank transfer) payment, any amount | ☐ ☐ | ☐ ☐ | v1 — **must pass** |
| T8 | Failed card charge returns a clear error | ☐ ☐ | ☐ ☐ | v1 (failure handling) |
| T9 | Webhooks arrive for order, payment, fulfillment | ☐ ☐ | ☐ ☐ | v1 — **must pass** |
| T10 | Theme shows the deposit option (Dawn) | ☐ ☐ | ☐ ☐ | v1 — **must pass** |
| T11 | Which scopes need Shopify approval, and lead time | — | — | v1 planning |
| T12 | After uninstalling the test app, is the deposit option removed from the product? | ☐ Removed ☐ Stays | ☐ Removed ☐ Stays | v1 uninstall cleanup |

---

## Part A — Set up (about 1 hour)

### A1. Partner account and development stores
1. Create a free Shopify Partner account at **partners.shopify.com**.
2. In the Partner (or Dev) Dashboard → **Stores** → **Add store** → **Development store**.
3. Create **two** stores:
   - `deposit-test-basic` — choose a **non-Plus** plan (Basic or Grow) if the plan choice is offered.
   - `deposit-test-plus` — choose **Plus** if offered.
   - If no plan choice is offered, create one store, then ask Shopify Partner Support how to test Plus-only features. Record which plan each store reports (query in A4).

### A2. Test payments (each store)
**Settings → Payments**, then either:
- **Shopify Payments → Enable test mode** (preferred — closest to real saved cards), or
- **Add payment method → (for testing) Bogus Gateway**.

| Test card (Shopify Payments test mode) | Result |
|---|---|
| `4242 4242 4242 4242` | Success |
| `4000 0000 0000 0002` | Declined |

Any future expiry, any CVV, any name.
Bogus Gateway: card number `1` = success, `2` = failure, `3` = error.

### A3. Test product (each store)
- Product: **Oak Dining Table**, price **$1,000**, inventory tracked, quantity 100.
- Theme: **Dawn** (default) — keep it.
- **Online Store → Preferences:** remove the password page or note the password so you can buy as a customer.

### A4. Throw-away test app with GraphiQL (about 20 minutes)
You need an app (not just the admin) because saved-card charges and selling plans are app-only APIs.

```bash
npm install -g @shopify/cli@latest
```
```bash
shopify app init --name deposit-feasibility
```
(choose the minimal / no-UI template if offered)

Edit `shopify.app.toml` → `[access_scopes]`:

```toml
[access_scopes]
scopes = "read_products,write_products,read_purchase_options,write_purchase_options,read_orders,write_orders,read_payment_terms,read_customers,read_payment_mandate,write_payment_mandate,read_fulfillments"

[webhooks]
api_version = "2026-10"

  [[webhooks.subscriptions]]
  topics = ["orders/create", "orders/updated", "order_transactions/create", "fulfillments/create"]
  uri = "/webhooks"
```

Run it against the non-Plus store:

```bash
cd deposit-feasibility
```
```bash
shopify app dev
```

- Select `deposit-test-basic` when asked, install the app.
- Press **`g`** in the terminal to open **GraphiQL** with the app's permissions. All queries below run there.
- If a scope is refused (e.g. payment mandate scopes), note it in **T11** and request access in the Partner Dashboard → your app → **API access**.

Check the store plan:

```graphql
{ shop { name currencyCode plan { displayName partnerDevelopment shopifyPlus } } }
```
Record `displayName` and `shopifyPlus` for each store.

> Field and enum names below follow the current Admin GraphQL API. If GraphiQL reports an unknown field, open the docs panel in GraphiQL, find the closest name and note the difference in your results — that is useful information for development.

---

## Part B — Optional no-code check (30 minutes)

Before writing any GraphQL, install an existing deposit/pre-order app that has a free plan on `deposit-test-basic`, set a 30% deposit on the Oak Dining Table, and buy it with a test card.

- If the order shows **Partially paid** with one order, and the app can collect the balance → strong signal the platform supports the flow on non-Plus.
- Uninstall it afterwards so it doesn't interfere with your tests.

This doesn't replace Part C (you need to know *your* app can do it), but it tells you within 30 minutes whether to continue.

---

## Part C — The tests (run on the non-Plus store first, then repeat on Plus)

Get the product id first:

```graphql
{ products(first: 5, query: "title:'Oak Dining Table'") { nodes { id title } } }
```

### T1 — Create the deposit option (selling plan)

```graphql
mutation CreateDepositPlan($input: SellingPlanGroupInput!, $resources: SellingPlanGroupResourceInput) {
  sellingPlanGroupCreate(input: $input, resources: $resources) {
    sellingPlanGroup { id name sellingPlans(first: 1) { nodes { id name } } }
    userErrors { field message code }
  }
}
```
Variables (replace the product id):
```json
{
  "input": {
    "name": "Pay 30% deposit now, rest later",
    "merchantCode": "deposit-test-30",
    "options": ["Deposit"],
    "sellingPlansToCreate": [{
      "name": "Pay 30% deposit now, rest later",
      "options": "30% deposit",
      "category": "PRE_ORDER",
      "billingPolicy": { "fixed": {
        "checkoutCharge": { "type": "PERCENTAGE", "value": { "percentage": 30 } },
        "remainingBalanceChargeTrigger": "TIME_AFTER_CHECKOUT",
        "remainingBalanceChargeTimeAfterCheckout": "P90D"
      }},
      "deliveryPolicy": { "fixed": { "fulfillmentTrigger": "UNKNOWN" } },
      "inventoryPolicy": { "reserve": "ON_SALE" }
    }]
  },
  "resources": { "productIds": ["gid://shopify/Product/REPLACE_ME"] }
}
```

**Pass:** no `userErrors`; a `sellingPlanGroup.id` is returned.
**Record:** any enum the API rejected and what it accepted instead.

### T10 — Theme shows the deposit option
1. Open the product page on the storefront.
2. **Pass:** Dawn shows *One-time purchase* / *Pay 30% deposit now, rest later* (or similar) options.
3. Select the deposit option, quantity **10** ($10,000), add to cart.

### T2 — Deposit checkout creates ONE partly-paid order
1. Checkout as a customer with email `buyer+1@example.com` and card `4242 4242 4242 4242`.
2. **Check at checkout:** total $10,000, due today **$3,000**, due later **$7,000** (record the exact wording Shopify shows).
3. Tick "save card / agree to future charge" if Shopify shows such a box (record it).
4. After paying, in GraphiQL:

```graphql
{
  orders(first: 1, reverse: true) {
    nodes {
      id name displayFinancialStatus
      currentTotalPriceSet { shopMoney { amount currencyCode } }
      totalReceivedSet     { shopMoney { amount } }
      totalOutstandingSet  { shopMoney { amount } }
      paymentTerms { paymentTermsType dueInDays paymentSchedules(first: 5) { nodes { id dueAt completedAt } } }
      paymentCollectionDetails { additionalPaymentCollectionUrl vaultedPaymentMethods { id } }
      lineItems(first: 5) { nodes { name quantity sellingPlan { sellingPlanId name } } }
      transactions(first: 10) { id kind status gateway amountSet { shopMoney { amount } } }
    }
  }
}
```

**Pass:**
- `displayFinancialStatus` = `PARTIALLY_PAID`
- received **3000.00**, outstanding **7000.00**
- **Admin → Orders** shows only ONE new order.

**Record:** the order id, `paymentTerms`, and whether `additionalPaymentCollectionUrl` and `vaultedPaymentMethods` are filled.

### T3 — Saved card on the order
**Pass:** `paymentCollectionDetails.vaultedPaymentMethods` has at least one `id` (a payment mandate).
Fail → saved-card charging is impossible for this setup; v1 uses payment link only.

### T4 — Charge the whole balance on the saved card
Use a **new** test order from T2 for each charge test (keep one unpaid order for T6 and T7).

```graphql
mutation ChargeBalance($id: ID!, $mandateId: ID!, $key: String!) {
  orderCreateMandatePayment(id: $id, mandateId: $mandateId, idempotencyKey: $key, autoCapture: true) {
    job { id done }
    paymentReferenceId
    userErrors { field message code }
  }
}
```
```json
{ "id": "gid://shopify/Order/REPLACE", "mandateId": "gid://shopify/PaymentMandate/REPLACE", "key": "t4-order-REPLACE-1" }
```

Then check the result (repeat after ~10 seconds if pending):

```graphql
query Status($orderId: ID!, $ref: String!) {
  orderPaymentStatus(orderId: $orderId, paymentReferenceId: $ref) { status errorMessage }
}
```

Re-run the T2 order query.

**Pass:** status succeeds (e.g. `PURCHASED`/`CAPTURED`), order `PAID`, outstanding `0.00`, still ONE order.
**Also test idempotency:** run the same mutation again with the **same** `key` → **Pass** if no second charge appears in `transactions`.
**Record:** any scope/permission error text (→ T11).

### T5 — Charge only part of the balance (later phases)
New deposit order ($7,000 outstanding). Same mutation, add an amount:

```graphql
mutation ChargePart($id: ID!, $mandateId: ID!, $key: String!, $amount: MoneyInput) {
  orderCreateMandatePayment(id: $id, mandateId: $mandateId, idempotencyKey: $key, amount: $amount, autoCapture: true) {
    job { id } paymentReferenceId userErrors { field message code }
  }
}
```
```json
{ "id": "gid://shopify/Order/REPLACE", "mandateId": "gid://shopify/PaymentMandate/REPLACE", "key": "t5-order-REPLACE-1",
  "amount": { "amount": "4000.00", "currencyCode": "USD" } }
```

**Pass:** outstanding becomes **3000.00**, order stays `PARTIALLY_PAID`.
**Record exactly** what happens on non-Plus vs Plus — this decides whether 3–4 step plans can use cards on non-Plus.

### T6 — Balance payment link on the same order
1. Take an unpaid deposit order (outstanding $7,000).
2. Copy `paymentCollectionDetails.additionalPaymentCollectionUrl`.
3. Open it in a **private/incognito window**, pay with `4242 4242 4242 4242`.
4. Re-run the order query.

**Pass:** order `PAID`, outstanding `0.00`, and **Admin → Orders shows no new order**.
**Record:** what the buyer sees on that page (amount shown, can they change it?).

Fallback test if the URL is empty: in **Admin → the order → Collect payment → Send invoice** (or via `orderInvoiceSend`) — record whether that invoice pays the same order.

### T7 — Record a bank transfer (any amount)
Unpaid deposit order, outstanding $7,000. Record **$4,000** first:

```graphql
mutation Manual($id: ID!, $amount: MoneyInput!) {
  orderCreateManualPayment(id: $id, amount: $amount, paymentMethodName: "Bank transfer") {
    order { id displayFinancialStatus totalOutstandingSet { shopMoney { amount } } }
    userErrors { field message }
  }
}
```
```json
{ "id": "gid://shopify/Order/REPLACE", "amount": { "amount": "4000.00", "currencyCode": "USD" } }
```

**Pass:** outstanding **3000.00**, `PARTIALLY_PAID`. Then record **3000.00** → `PAID`.
**Record:** whether amount above outstanding is rejected (expected).

### T8 — Failed card charge
1. New deposit order paid with a card that saves but fails later charges. With Shopify Payments test mode try `4000 0000 0000 0341` (attaches, then declines later charges — confirm it is supported); with the Bogus Gateway, record whether a later failure can be simulated.
2. Run T4 on that order.

**Pass:** `orderPaymentStatus` returns an error status with a readable `errorMessage`; order stays `PARTIALLY_PAID`.
If no failing card can be simulated → record it; failure handling will be tested with mocked API responses instead.

### T9 — Webhooks
While `shopify app dev` runs, the terminal (or your `/webhooks` route log) shows incoming webhooks.

| Action | Expected webhook |
|---|---|
| T2 checkout | `orders/create`, `order_transactions/create` |
| T4 / T6 / T7 payments | `order_transactions/create`, `orders/updated` |
| Mark order fulfilled in admin | `fulfillments/create` |

**Pass:** each arrives within ~1 minute. **Record:** a sample payload of each (save to `fixtures/` — they become test fixtures in development).

### T11 — Scopes and approvals
**Record:**
- Scopes refused or marked "protected" (likely the payment mandate scopes).
- Where Shopify asks you to request access and the stated review time.

If approval takes weeks → start the request **now**, and build v1 with **payment link first**, saved-card charging behind a feature flag.

### T12 — Uninstall behaviour (run last on each store)
1. Note the product's `.js` page shows `selling_plan_groups` with the deposit option.
2. Uninstall the test app (**Settings → Apps → Uninstall**).
3. Reload `/products/oak-dining-table.js`.

**Record:** whether `selling_plan_groups` is now empty (Shopify removed the app's deposit option) or still lists it. If it stays, the spec needs a cleanup step before uninstall (e.g. a "Remove deposit options" button in Settings).

---

## Part D — Repeat on the Plus store
1. Stop `shopify app dev`, run it again and select `deposit-test-plus`.
2. Repeat **A3, T1–T10**.
3. Fill the Plus column of the summary table.

---

## Part E — Decide

| Result | Decision |
|---|---|
| T1, T2, T7, T9, T10 pass on both stores | ✅ **Start development** (the core works everywhere) |
| T1 or T2 fails on non-Plus | ⛔ **Stop.** The product idea needs re-planning (e.g. Plus-only app, or draft-order based) |
| T3/T4 fail on non-Plus, pass on Plus | v1: payment link only on non-Plus; saved card on Plus (`features.savedCardCharge`) |
| T4 blocked only by scope approval | Request access now; launch with link first, enable card when approved |
| T5 passes on non-Plus | 3–4 step plans with cards possible on every plan (Appendix C of the spec) |
| T5 fails on non-Plus | 3–4 step plans on non-Plus use bank transfer for middle steps |
| T6 fails (new order created or no link) | Use Shopify's invoice (`orderInvoiceSend`) for links; re-test |
| T8 can't be simulated | Test failures with mocked responses; verify on a live beta store later |

---

## What to hand to the developer afterwards
- The filled **summary table**.
- Any field/enum names that differed from the spec.
- Saved webhook payloads (`fixtures/`).
- Scope approval status and dates.

Then update `docs/deposit-app-spec.md` Module 0 and the plan-gated features in Module 7 to match the results.
