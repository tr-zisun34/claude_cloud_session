# Stagepay: B2B Deposit — B2B Feasibility Test Plan

Version 1.0 · 2026-11-04 · Extends `docs/feasibility-test-guide.md` (tests T1–T12)

**Purpose:** before redesigning Stagepay as a B2B app, prove how deposits and milestone payments can work for **B2B company buyers**:

- **Non-Plus stores** (Basic, Grow, Advanced): Shopify B2B has companies, payment terms and saved cards, but **no deposits**. Stagepay must provide the deposit.
- **Plus stores:** Shopify has a fixed deposit % per company location. Stagepay must add **milestones** (e.g. 30/40/30) on top of it.

No real money is used: development stores + test cards only.

**Time:** about 1 day (non-Plus ½ day, Plus ½ day).

---

## 0. The questions this plan answers

| # | Question | Decides |
|---|---|---|
| Q1 | Does the **deposit purchase option** (selling plan) work when a **logged-in B2B buyer** checks out? | Whether non-Plus B2B deposits can use the method already tested in T1–T4 |
| Q2 | Can the app charge **part** of the balance on a saved card? (T5) | Plus milestones (30/40/30); non-Plus fallback deposit |
| Q3 | If Q1 fails: can a B2B order on **payment terms** be turned into a deposit order by charging the deposit part right after the order? | Non-Plus fallback design |
| Q4 | Can Stagepay add milestones **on top of Shopify's native Plus deposit** on the same order? | Plus tier design |
| Q5 | Can the app read the **company, location, contact, PO number and payment terms** of an order? | Company-based rules, B2B emails, dashboard by company |
| Q6 | Do payment links and recorded bank transfers work on **B2B orders**? | Balance collection for B2B |
| Q7 | Do B2B **draft orders** (quotes) work with the deposit flow? | Quote-based ordering |

---

## 1. Results summary (fill in at the end)

| Test | What | Non-Plus | Plus | Needed for |
|---|---|---|---|---|
| B1 | B2B setup works (company, location, contact, terms) | ☐ Pass ☐ Fail | ☐ Pass ☐ Fail | Everything |
| B2 | App can read companies and order company data (Q5) | ☐ ☐ | ☐ ☐ | Company rules, emails, dashboard |
| T13 | Deposit purchase option in **B2B checkout** (Q1) | ☐ ☐ | ☐ ☐ | Non-Plus deposit method |
| T15 | B2B order on payment terms — what Shopify creates | ☐ ☐ | ☐ ☐ | Fallback design |
| T5 | Partial saved-card charge (Q2) | ☐ ☐ | ☐ ☐ | Milestones, fallback |
| T16 | Deposit charged right after a terms order (Q3) | ☐ ☐ | — | Non-Plus fallback |
| T17 | Recorded bank transfer (part amount) on a B2B order (Q6) | ☐ ☐ | ☐ ☐ | Balance collection |
| T18 | Payment link on a B2B order (Q6) | ☐ ☐ | ☐ ☐ | Balance collection |
| T14 | Native Plus deposit + Stagepay 40% + 30% (Q4) | — | ☐ ☐ | Plus milestones |
| T19 | Read native deposit % of a company location | — | ☐ ☐ | Plus: avoid double deposits |
| T20 | B2B draft order (quote) + deposit (Q7) | ☐ ☐ | ☐ ☐ | Quote flow |
| T21 | Webhooks for B2B orders | ☐ ☐ | ☐ ☐ | Order sync |

---

## 2. Setup

### 2.1 Stores
| Store | Plan | Notes |
|---|---|---|
| `test-store-1-epmpstlq` (existing) | Non-Plus | Already has the Oak Dining Table and deposit option `6174703706` |
| Plus dev store | Plus | Create it (feasibility guide A1) if not done; add the Oak Dining Table ($1,000) |

Keep **one** Stagepay deposit option per product (delete leftovers: `sellingPlanGroupDelete`).

### 2.2 Test app scopes
Add the B2B scopes to `shopify.app.toml` of the throw-away test app, then restart `shopify app dev` and accept:

```toml
scopes = "read_products,write_products,read_purchase_options,write_purchase_options,read_orders,write_orders,read_payment_terms,read_customers,read_companies,write_companies,read_draft_orders,write_draft_orders,read_payment_mandate,write_payment_mandate,read_fulfillments"
```

⚠ Record any scope that is refused or needs approval (add to T11).

### 2.3 Turn on B2B and customer accounts (each store)
1. **Settings → Customer accounts →** use **new customer accounts** (B2B buyers log in with them).
2. **Customers → Companies** must be visible. If not, look for **Settings → B2B** (or Shopify's B2B setup prompt) and turn B2B on.
3. **Settings → Payments → Payment terms:** enable payment terms if offered (needed for Net 30 / Due on fulfillment).
4. Record where each setting was found and any plan limits shown.

### 2.4 Create the test company (each store)
**Customers → Companies → Add company**

| Field | Value |
|---|---|
| Company name | Restaurant ABC |
| Location name | Restaurant ABC – Main |
| Billing / shipping address | any valid address in the store's country |
| Contact (customer) | `buyer-b2b+1@example.com` (a new customer), role *Ordering only* or *Location admin* |
| Catalog | none needed (default prices) — optional: a catalog with 10% off to check prices |

**Location payment settings** (open the location → *Payment terms* / *Checkout*):

| Store | Setting for test |
|---|---|
| Non-Plus | Run the tests twice: (a) **No payment terms** (pay at checkout), (b) **Due on fulfillment** or **Net 30** |
| Plus | (a) **No terms**, (b) terms + **Require deposit 30%** (for T14) |

Also on each location: allow **saving / vaulting credit cards** if the option exists.

Create a **second company** later if needed: *Hotel Group XYZ* with **Net 30**.

### 2.5 Log in as the B2B buyer
1. Open the storefront in a **private window**.
2. **Account → Log in** with `buyer-b2b+1@example.com` (code sent by email to the test inbox; for `example.com` addresses use a real test inbox you own instead, e.g. `yourname+b2b@gmail.com`).
3. Choose the location **Restaurant ABC – Main** if asked.
4. **Check:** the storefront now behaves as a B2B buyer (catalog prices if a catalog is set, B2B checkout).

---

## 3. Tests

Run every test on the **non-Plus store first**, then on Plus where the table says so. Paste results into section 1.

### B1 — B2B setup works
**Pass:** company, location and contact exist; buyer can log in and choose the location; checkout shows the company name/location.

### B2 — App can read company data (Q5)
In GraphiQL:

```graphql
{
  companies(first: 5) {
    nodes {
      id name externalId
      locations(first: 5) {
        nodes {
          id name
          billingAddress { address1 city countryCode }
          buyerExperienceConfiguration {
            checkoutToDraft
            paymentTermsTemplate { id name paymentTermsType dueInDays }
          }
        }
      }
      contacts(first: 5) { nodes { id customer { id email } } }
    }
  }
}
```

**Pass:** Restaurant ABC, its location, payment terms and the contact's email are returned.
⚠ Record field names that differ.

### T13 — Deposit purchase option in B2B checkout (Q1)
Logged in as the B2B buyer (2.5), location with **no payment terms**:

1. Open `/products/oak-dining-table.js` **while logged in**. Record whether `selling_plan_groups` still lists the deposit option.
2. Add with the deposit:
   ```
   https://test-store-1-epmpstlq.myshopify.com/cart/add?id=67952566992986&quantity=10&selling_plan=6174703706
   ```
   If that link drops the plan, use the browser-console `fetch('/cart/add.js', …)` method from the feasibility guide.
3. Open the cart. Record: is the line still showing "Pay 30% deposit now, rest later"? Any warning or error?
4. Check out. Record what checkout shows: full price or **due today $3,000 / due later $7,000**? Any error?
5. If it completes, run the **B2B order query** (below) and record the results.
6. Repeat 2–5 with the location on **payment terms** (Net 30 / Due on fulfillment).

**B2B order query** (used in many tests):

```graphql
{
  orders(first: 1, reverse: true) {
    nodes {
      id name displayFinancialStatus poNumber
      purchasingEntity {
        ... on PurchasingCompany {
          company { id name }
          location { id name }
          contact { id customer { email } }
        }
      }
      currentTotalPriceSet { shopMoney { amount currencyCode } }
      totalReceivedSet     { shopMoney { amount } }
      totalOutstandingSet  { shopMoney { amount } }
      paymentTerms { paymentTermsType dueInDays paymentSchedules(first: 5) { nodes { id dueAt completedAt amount { amount } } } }
      paymentCollectionDetails { additionalPaymentCollectionUrl vaultedPaymentMethods { id } }
      lineItems(first: 5) { nodes { name quantity sellingPlan { sellingPlanId name } } }
      transactions(first: 10) { id kind status gateway amountSet { shopMoney { amount } } }
    }
  }
}
```

| Result | Meaning |
|---|---|
| ONE order, `PARTIALLY_PAID`, 3,000 / 7,000, `purchasingEntity` = Restaurant ABC, mandate present | ✅ **Non-Plus B2B deposits can use selling plans** (current design) |
| Plan dropped from cart, or checkout charges the full price, or an error | ❌ Selling plans don't work for B2B buyers → run **T15 + T16** (fallback) |

Also run T13 on the **Plus** store (no native deposit set) and record.

### T15 — B2B order on payment terms: what Shopify creates
Location on **Due on fulfillment** (or Net 30). Logged in as the B2B buyer, add 10 tables **without** a selling plan:
```
https://test-store-1-epmpstlq.myshopify.com/cart/add?id=67952566992986&quantity=10
```
Check out (choose "pay later"/terms if offered; tick **save card** if offered). Enter PO number `PO-1001` if the field exists.

Run the **B2B order query**. Record:
- `displayFinancialStatus` (expected `PENDING` / payment due later)
- received / outstanding (expected 0 / 10,000)
- `paymentTerms` type and due date
- `poNumber`
- whether `vaultedPaymentMethods` has a card (needed for T16)

### T5 — Partial saved-card charge (Q2)
On an order that has a saved card and an outstanding balance (the T13 order, or a new retail deposit order from the feasibility guide):

```graphql
mutation ChargePart($id: ID!, $mandateId: ID!, $key: String!, $amount: MoneyInput) {
  orderCreateMandatePayment(id: $id, mandateId: $mandateId, idempotencyKey: $key, amount: $amount, autoCapture: true) {
    job { id done } paymentReferenceId userErrors { field message code }
  }
}
```
```json
{ "id": "ORDER_ID", "mandateId": "MANDATE_ID", "key": "t5-partial-1",
  "amount": { "amount": "4000.00", "currencyCode": "USD" } }
```

Wait ~10 s, then run the order query.

| Result | Meaning |
|---|---|
| Outstanding drops by 4,000, still `PARTIALLY_PAID` | ✅ Partial charges allowed on this plan |
| `userErrors` mentioning amount / plan | ❌ Not allowed on this plan — record the exact message |

Run on **non-Plus and Plus**.

### T16 — Deposit right after a terms order (non-Plus fallback, Q3)
Only needed if **T13 failed** on non-Plus. Use the **T15 order** (10,000 due, saved card present).

1. Charge **3,000** with the T5 mutation (`key: "t16-deposit-1"`).
   - Pass → order shows 3,000 received, 7,000 outstanding → the fallback deposit works by card.
2. If T5 failed on non-Plus: record **3,000** as a manual payment (T17) instead.
   - This proves the deposit can at least be tracked, but the buyer must pay it by bank transfer or link.
3. If there's **no saved card** on terms orders: record it — the fallback then needs a payment link (T18).

### T17 — Recorded bank transfer on a B2B order (Q6)
On any B2B order with an outstanding balance:

```graphql
mutation Manual($id: ID!, $amount: MoneyInput!) {
  orderCreateManualPayment(id: $id, amount: $amount, paymentMethodName: "Bank transfer") {
    order { id displayFinancialStatus totalOutstandingSet { shopMoney { amount } } }
    userErrors { field message }
  }
}
```
```json
{ "id": "ORDER_ID", "amount": { "amount": "3000.00", "currencyCode": "USD" } }
```

**Pass:** outstanding drops by exactly 3,000; order stays partly paid.

### T18 — Payment link on a B2B order (Q6)
1. Copy `additionalPaymentCollectionUrl` from a B2B order with a balance.
2. Open it in a **new private window** (not logged in).
3. Record: does it ask the buyer to **log in**? What amount does it show (whole balance only, or can it be a part)? Pay with `4242 4242 4242 4242`.
4. Run the order query.

**Pass:** payment lands on the **same** order, no new order. Record the amount it collected.

### T14 — Plus: native deposit + Stagepay milestones (Q4)
Plus store, location payment terms with **Require deposit 30%** (2.4).

1. Logged in as the B2B buyer, add 10 tables **without** a selling plan and check out.
   **Expected:** Shopify takes **3,000** at checkout (native deposit), 7,000 outstanding.
2. Run the B2B order query: record status, received, outstanding, `paymentSchedules`, mandate.
3. **Milestone 2:** charge **4,000** with the T5 mutation (`key: "t14-m2"`). Expected: outstanding 3,000.
4. **Milestone 3:** charge the remaining **3,000** — either with `amount` 3,000 or **without** `amount` (whole remaining). Expected: `PAID`.
5. Record: did Shopify's own payment schedule (due date) conflict with the extra charges? Any error?

**Pass:** one order goes 3,000 → 7,000 → 10,000 paid in three payments.

Also try: same company, product **with** the Stagepay selling plan while the native 30% deposit is on. Record whether Shopify combines, blocks or doubles the deposit (Stagepay must avoid double deposits).

### T19 — Read a location's native deposit % (Plus)
```graphql
{
  companyLocation(id: "gid://shopify/CompanyLocation/REPLACE") {
    name
    buyerExperienceConfiguration {
      paymentTermsTemplate { name paymentTermsType dueInDays }
      deposit { ... on DepositPercentage { percentage } }
    }
  }
}
```
⚠ Verify field names (`deposit`, `DepositPercentage`) in the GraphiQL docs panel.
**Pass:** returns `percentage: 30`. Stagepay can then show "Shopify deposit 30% + Stagepay milestones 40/30" and never add a second deposit.

### T20 — B2B draft order (quote) + deposit (Q7)
1. **Orders → Drafts → Create order**, customer = the B2B contact, choose **Restaurant ABC – Main**, add 10 tables.
2. Record the payment options shown:
   - Non-Plus: payment terms only? any deposit field?
   - Plus: **Require deposit** field (expected Plus-only).
3. **Send invoice** to the buyer → pay as the buyer.
4. Run the order query on the resulting order: status, received, outstanding, mandate.
5. Non-Plus: try the T16 approach on this order (charge or record 3,000).

### T21 — Webhooks for B2B orders
With `shopify app dev` running, place one B2B order (T13 or T15). Save the `orders/create` payload to `tests/fixtures/b2b-orders-create.json`.
**Record:** does the payload include `company` / `company_location` / `po_number` fields, or must the app query them (B2)?

---

## 4. Decision matrix

| Results | Non-Plus design (Stagepay B2B Deposit) |
|---|---|
| **T13 ✅** | Keep the current method: deposit purchase option, per-company rules decide who sees it. Fastest path. |
| **T13 ❌, T15 + T16 card ✅** | **Terms-order method:** buyer orders on payment terms; Stagepay charges the deposit on the saved card right after the order, then the balance later. Needs partial charge on non-Plus (T5 ✅). |
| **T13 ❌, T5 ❌ on non-Plus** | Deposit by **payment link / bank transfer** only (T17, T18). Weaker product — review pricing and positioning before building. |
| **T13 ❌, no saved card on terms orders** | Link + bank transfer for the deposit; card only if the buyer saves one later. |

| Results | Plus design (Stagepay B2B Milestones) |
|---|---|
| **T14 ✅, T19 ✅** | Build milestones on top of the native deposit: Shopify takes step 1, Stagepay collects steps 2…N by card / link / bank transfer. |
| **T14 ❌ (extra charges blocked)** | Plus milestones use **links + bank transfer** for middle steps; revisit tier value. |
| **T19 ❌** | Merchant enters the native deposit % in Stagepay manually; app warns about double deposits. |

| Results | Shared design |
|---|---|
| **B2 ✅** | Company-based deposit terms, emails to company contacts, dashboard by company |
| **T18 shows login required** | Emails explain "log in to your company account to pay"; offer bank transfer details too |
| **T20 ✅** | Quote → deposit flow in v1; otherwise later phase |

---

## 5. After testing

1. Fill in section 1 and send it with: changed field names, refused scopes, saved webhook payloads, screenshots of B2B checkout.
2. Then update, in this order:
   - `docs/deposit-app-spec.md` — B2B design (company terms, Plus milestones, chosen non-Plus method)
   - `docs/stagepay-development-guide.md` — new and changed steps
   - `Build_Order_Estimate.xlsx` — new hours (expected +40 to 60 h)
