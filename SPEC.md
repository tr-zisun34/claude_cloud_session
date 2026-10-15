# Cart Sentinel — Build Specification

> Working name: **Cart Sentinel**. Renaming later only touches the app listing, the `cs` prefixes and the app proxy path.

A Shopify app with two layers:

1. **Bot Block** — stops automated browsers, scraper extensions and suspicious networks while people browse the store.
2. **Rules Engine** — merchant-built IF/THEN rules (AND / OR) that hide payment methods at checkout and act on orders after they're placed (hold fulfillment, cancel, tag, email notification). Rules run in list order; the first match wins.

Admin pages: **Dashboard**, **Rules**, **Orders**, **Bot Protection**.

This document is the single source of truth for the build. It is split into 16 features. **Each feature is one branch and one PR**, built and tested before the next starts. Total estimate: **62.5 hours** (see `Store_Shield_Estimation.xlsx`).

---

## Contents

- [0. Ground rules](#0-ground-rules)
  - [0.1 Scope](#01-scope)
  - [0.2 Tech stack](#02-tech-stack)
  - [0.3 Repo layout](#03-repo-layout)
  - [0.4 Conventions](#04-conventions)
  - [0.5 Shopify setup before F01](#05-shopify-setup-before-f01)
  - [0.6 Environment variables](#06-environment-variables)
  - [0.7 How the pieces fit](#07-how-the-pieces-fit)
  - [0.8 Feature order](#08-feature-order)
- [F01 Database setup and schema](#f01--database-setup-and-schema)
- [F02 App proxy and visitor check](#f02--app-proxy-and-visitor-check)
- [F03 Store script (theme extension)](#f03--store-script-theme-extension)
- [F04 Automation and extension checks](#f04--automation-and-extension-checks)
- [F05 IP intelligence](#f05--ip-intelligence)
- [F06 Bot Protection page](#f06--bot-protection-page)
- [F07 Rule engine core](#f07--rule-engine-core)
- [F08 Rule conditions and order intake](#f08--rule-conditions-and-order-intake)
- [F09 Checkout function](#f09--checkout-function)
- [F10 Order actions](#f10--order-actions)
- [F11 Rules page and rule builder](#f11--rules-page-and-rule-builder)
- [F12 Orders page](#f12--orders-page)
- [F13 Dashboard](#f13--dashboard)
- [F14 Privacy and uninstall](#f14--privacy-and-uninstall)
- [F15 Pricing plan](#f15--pricing-plan)
- [F16 Go live](#f16--go-live)
- [Appendix A — Condition catalog](#appendix-a--condition-catalog)
- [Appendix B — Action catalog](#appendix-b--action-catalog)
- [Appendix C — Tags, attributes and names](#appendix-c--tags-attributes-and-names)
- [Appendix D — Open questions](#appendix-d--open-questions)

---

## 0. Ground rules

### 0.1 Scope

**In scope (v1)**
- Bot checks on every storefront page: automation tools, scraper extensions, data-centre / VPN / Tor traffic, blocked networks (ASN) and countries, page-speed limit.
- Each bot check set to **Off / Monitor / Block** by the merchant.
- Two ordered rule lists: **Checkout rules** and **Post-order rules**. Rules run top to bottom and **the first rule that matches is applied**; the rest are skipped.
- Each rule has a name, **AND / OR** conditions and one main action:
  - **Checkout:** Allow or Hide payment methods (e.g. COD).
  - **Post-order:** Allow, Hold fulfillment or Cancel (refund / restock / notify customer), plus optional Tag and Email notification.
- Rules can be enabled or disabled and moved up or down. Post-order rules can also run in **test mode** (log only).
- Orders page with review actions: Approve, Cancel, Mark as fraud.
- Fraud list (scrambled email / phone / address) used by the "past fraud" condition.
- Allowlist (IP, device, email, customer).
- Shopify privacy webhooks, data retention, pricing plan, hosting.

**Out of scope (v1)**
- Showing fake data to scrapers.
- Blocking plain HTTP scrapers that never run JavaScript. No Shopify app can intercept storefront requests; Shopify handles DDoS itself.
- Machine-learning scoring.
- Hiding the "Buy it now" button on product pages.

### 0.2 Tech stack

| Layer | Choice | Notes |
|---|---|---|
| App | Shopify CLI app template (React Router v7, formerly Remix) + TypeScript | Use whatever template `shopify app init` currently generates. |
| Admin UI | The UI components the template ships with (Polaris) + App Bridge | Embedded in Shopify admin. |
| Database | PostgreSQL via Prisma | Local Postgres in Docker from day one. |
| Storefront | Theme app extension (**app embed block**) + vanilla JS | No framework. Script ≤ 15 KB minified. |
| Checkout | Shopify Function — payment customization (JavaScript) | Separate extension under `extensions/`. |
| IP data | MaxMind GeoLite2 (ASN + Country), cloud provider IP lists, Tor exit list | Free. Files on disk, refreshed by jobs. |
| Email | Nodemailer over SMTP; **Mailpit** locally | |
| Tests | Vitest (unit), Playwright (bot tests), `shopify app function run` (function) | |
| Jobs | `node-cron` inside the app process | Fine for one instance in v1. |

### 0.3 Repo layout

```
app/
  routes/
    app._index.tsx              # Dashboard (F13)
    app.rules._index.tsx        # Rules list (F11)
    app.rules.$id.tsx           # Rule builder (F11)
    app.orders._index.tsx       # Orders list (F12)
    app.orders.$id.tsx          # Order detail (F12)
    app.bot-protection.tsx      # Bot Protection (F06)
    proxy.check.tsx             # Visitor check via app proxy (F02)
    webhooks.*.tsx              # Webhooks (F08, F14)
  bot/                          # decide(), detectors' server side (F02, F04, F05)
  ip/                           # geo, ranges, hosting ASNs, good bots, speed (F05)
  rules/                        # types, catalog, evaluate, rules.server, templates (F07, F08, F11)
  orders/                       # intake, context builder, actions (F08, F10, F12)
  services/                     # shopify graphql helpers, email, hashing, payment customization
  jobs/                         # cron jobs (F05, F14)
  db.server.ts
extensions/
  cs-storefront/                # theme app extension: app embed + sentinel.js (F03, F04)
  cs-payment-rules/             # payment customization function (F09)
prisma/
  schema.prisma
  migrations/
tests/
  unit/                         # vitest
  bots/                         # playwright bot scripts (F04, F05)
  fixtures/                     # shared rule + context fixtures (F07, F09)
data/                           # downloaded IP files (gitignored)
docker-compose.yml              # postgres + mailpit (F01)
SPEC.md
```

### 0.4 Conventions

- **One feature = one branch = one PR.** Branch `feat/fNN-short-name`; PR title `FNN: <feature name>`.
- **Tables are created in the feature that first needs them.** Each PR adds its own Prisma migration. Never edit a merged migration.
- **Every table has a `shop` column** (the `*.myshopify.com` domain). Every query filters by `shop`. No query may read across shops.
- **Names:** tags start `cs-`, cart attributes start `_cs_` (the underscore hides them from the customer), metafield namespace `$app:cart-sentinel`.
- **Fail open.** If any check, lookup or job fails, the visitor is allowed and the order is left alone. Log the error. The app must never take a store down.
- **No raw personal data in logs.** Log order IDs and hashes, not emails, phones or addresses.
- **Shopify API version:** pin the version in `shopify.app.toml` and check field names against that version's docs. GraphQL field names in this spec are a guide, not gospel.

**Every PR must include**
- [ ] Migration (if the feature adds or changes tables)
- [ ] Unit tests for new logic, all green (`npm test`)
- [ ] Typecheck and lint green
- [ ] Manual test steps from this spec done on the dev store, noted in the PR description
- [ ] No secrets committed; new env vars added to `.env.example`

### 0.5 Shopify setup before F01

1. Shopify Partner account and a **development store**.
2. **Protected customer data access** requested in the Partner Dashboard for: name, email, phone, address. Without it the app can't read those fields from orders. Level 1 is enough for development stores; public release needs approval.
3. **Scopes** (put in `shopify.app.toml`; confirm exact names in the current docs):
   `read_orders, write_orders, read_customers, write_customers, read_products, read_payment_customizations, write_payment_customizations, read_merchant_managed_fulfillment_orders, write_merchant_managed_fulfillment_orders`
4. **App proxy:** prefix `apps`, subpath `cs`, URL `https://<app-url>/proxy`. The storefront then calls `/apps/cs/check`.
5. **Free MaxMind account** + license key (for GeoLite2 downloads).
6. In the dev store: enable a **manual payment method named "Cash on Delivery (COD)"** and the **Bogus Gateway** for test card payments.

### 0.6 Environment variables

| Name | Used by | Example |
|---|---|---|
| `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `SCOPES`, `SHOPIFY_APP_URL` | Template | set by CLI |
| `DATABASE_URL` | Prisma | `postgresql://cs:cs@localhost:5432/cs` |
| `HASH_SECRET` | Fraud list hashing | 32+ random bytes, never changes after launch |
| `MAXMIND_ACCOUNT_ID`, `MAXMIND_LICENSE_KEY` | F05 | |
| `DATA_DIR` | F05 IP files | `./data` |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `ALERT_FROM_EMAIL` | F10 | Mailpit: `localhost`, `1025` |
| `CS_DEV_IP_OVERRIDE` | F05 testing | `true` only in development |

### 0.7 How the pieces fit

```
 Shopper's browser                         Our app server                       Shopify
 ─────────────────                         ──────────────                       ───────
 sentinel.js (F03/F04)
   collect signals ──POST /apps/cs/check──► proxy.check (F02)
                         (app proxy)          + IP checks (F05)
                                              + allowlist (F06)
                                              decide → allow / monitor / block
   ◄──────────── action, flags ───────────    save Visit
   block → overlay
   monitor → cart attribute _cs_flags ───────────────────────────────────────►  cart

 Checkout                                                                      Payment customization
   ────────────────────────────────────────────────────────────────────────►  function (F09) reads
                                                                               cart + config metafield
                                                                               → hides COD / methods
                                           rules saved (F11) ──metafield──►   (config written by server)

 Order placed ────────────────────────────────────────────────────────────►  orders/create webhook
                                           order intake (F08) ◄─────────────
                                           build context, evaluate rules
                                           save OrderCheck
                                           run actions (F10) ──GraphQL───►    hold / cancel / tag
                                           email alert (F10)
 Staff (admin)                             Orders page (F12) ──GraphQL───►    release hold / cancel / tag
                                           Mark as fraud → FraudEntry
```

### 0.8 Feature order

| # | Feature | Hours | Depends on | New tables |
|---|---|---|---|---|
| F01 | Database setup and schema | 3.5 | — | `Shop`, `Settings` |
| F02 | App proxy and visitor check | 2.5 | F01 | `Visit` |
| F03 | Store script (theme extension) | 3.5 | F02 | — |
| F04 | Automation and extension checks | 3.5 | F03 | — |
| F05 | IP intelligence | 5.0 | F02 | — |
| F06 | Bot Protection page | 4.0 | F02–F05 | `AllowlistEntry` |
| F07 | Rule engine core | 3.5 | F01 | `Rule` |
| F08 | Rule conditions and order intake | 6.5 | F05, F07 | `OrderCheck`, `FraudEntry` |
| F09 | Checkout function | 5.5 | F07 | — |
| F10 | Order actions | 5.0 | F08 | — |
| F11 | Rules page and rule builder | 7.0 | F07–F10 | — |
| F12 | Orders page | 4.5 | F10 | `StaffAction` |
| F13 | Dashboard | 2.5 | F02, F08 | — |
| F14 | Privacy and uninstall | 2.0 | all tables | — |
| F15 | Pricing plan | 2.0 | F08 | — |
| F16 | Go live | 2.0 | all | — |
| | **Total** | **62.5** | | |

> F08 now also covers **order intake** (receiving the order and saving the check). That's where the "past fraud" and "repeat orders" conditions get their data. F10 then only does the actions.

---

## F01 · Database setup and schema

**Hours:** 3.5 · **Branch:** `feat/f01-database` · **Depends on:** —

### Goal
App created from the Shopify template, running on the dev store, with local Postgres and the first two tables. Settings get default values on install.

### Database

```prisma
// prisma/schema.prisma — datasource switched from SQLite to PostgreSQL
model Session { /* unchanged from template */ }

model Shop {
  domain        String    @id                 // "example.myshopify.com"
  installedAt   DateTime  @default(now())
  uninstalledAt DateTime?
  plan          String?                       // F15
  settings      Settings?
}

model Settings {
  shop             String   @id
  shopRef          Shop     @relation(fields: [shop], references: [domain], onDelete: Cascade)
  // F02–F06 bot checks: each "OFF" | "MONITOR" | "BLOCK"
  botChecks        Json     @default("{\"automation\":\"BLOCK\",\"extension\":\"BLOCK\",\"datacenter\":\"MONITOR\",\"tor\":\"BLOCK\",\"speed\":\"BLOCK\",\"asn\":\"BLOCK\",\"country\":\"BLOCK\"}")
  blockedAsns      Int[]    @default([])
  blockedCountries String[] @default([])     // ISO-2 codes
  speedLimitPerMin Int      @default(60)      // page checks per IP per minute
  extensionSigs    Json     @default("[]")    // F04 known extension markers
  contactEmail     String?                    // shown on the block screen
  alertEmail       String?                    // F10 rule alerts
  updatedAt        DateTime @updatedAt
}
```

### Backend
- `docker-compose.yml` with `postgres:16` and `axllent/mailpit` (SMTP 1025, UI 8025).
- `app/services/settings.server.ts`
  - `getSettings(shop)` — returns settings; creates the `Shop` + default `Settings` row if missing.
  - `updateSettings(shop, patch)` — validated with Zod.
- Install hook (`afterAuth` in the template): call `getSettings(shop)` so the rows exist.

### Frontend
- App home shows "Cart Sentinel is installed" and the current settings as read-only JSON (temporary; replaced by the Dashboard in F13).

### Tests
- **Unit:** `getSettings` creates defaults once; second call returns the same row; `updateSettings` rejects an invalid mode like `"ON"`.
- **Manual:** `docker compose up -d`, `npm run dev`, install on the dev store, open the app, see the defaults. Check the rows in `npx prisma studio`.

### Done when
- [ ] App opens inside the dev store admin
- [ ] Postgres migration applied; `Shop` and `Settings` rows created on install
- [ ] Unit tests green

---

## F02 · App proxy and visitor check

**Hours:** 2.5 · **Branch:** `feat/f02-visitor-check` · **Depends on:** F01

### Goal
The storefront can call our server safely through Shopify's app proxy. The server reads the real visitor IP, decides allow / monitor / block from the flags it gets, and saves a `Visit`.

### Database

```prisma
enum VisitAction { ALLOW MONITOR BLOCK }

model Visit {
  id        BigInt      @id @default(autoincrement())
  shop      String
  deviceId  String
  ip        String
  asn       Int?
  asnOrg    String?
  country   String?
  path      String
  flags     String[]    // e.g. ["automation","datacenter"]
  action    VisitAction
  createdAt DateTime    @default(now())

  @@index([shop, createdAt])
  @@index([shop, deviceId])
  @@index([shop, ip])
}
```

### Backend
- **Route** `app/routes/proxy.check.tsx` (`POST`), reached at `https://<store>/apps/cs/check`.
  - Verify the app proxy signature with the template's `authenticate.public.appProxy(request)`. Reject unsigned requests with 401.
  - **Real IP:** read the visitor IP from the forwarded headers Shopify sends. **First task of this PR:** log the headers once and confirm the real visitor IP is there.
    - **If it isn't**, switch to a direct endpoint `POST https://<app-url>/api/check` with CORS limited to the shop's domains, and verify the shop with a short-lived signed token printed into the app embed by Liquid. Note the outcome in the PR.
  - **Request body:**
    ```json
    { "deviceId": "uuid", "path": "/products/x", "signals": { } }
    ```
  - **Response:**
    ```json
    { "action": "ALLOW" | "MONITOR" | "BLOCK", "flags": ["..."], "ref": "V-12345" }
    ```
    `ref` is the Visit ID, shown on the block screen so the merchant can find and unblock it.
- **`app/bot/decide.ts`** (pure function)
  - `decide(flags: Flag[], botChecks: Record<Check, Mode>): VisitAction`
  - Each flag maps to a check (e.g. `datacenter → datacenter`, `asn_blocked → asn`). If any matching check is `BLOCK` the result is `BLOCK`; else if any is `MONITOR` the result is `MONITOR`; otherwise `ALLOW`.
  - Special flags: `good_bot` and `allowlisted` always give `ALLOW`.
- In this PR, flags come only from a stub. Real flags arrive in F04 (browser) and F05 (IP).
- Wrap everything in try/catch. On error return `ALLOW` with HTTP 200.
- Save the `Visit` without making the response wait (fire and forget, errors logged).

### Frontend
None (the storefront script comes in F03).

### Tests
- **Unit:** `decide()` truth table: no flags → ALLOW; monitor-only flag → MONITOR; block + monitor → BLOCK; `good_bot` with block flags → ALLOW; check set to `OFF` ignores its flag.
- **Manual:** from the browser console on the storefront, `fetch('/apps/cs/check', {method:'POST', body: JSON.stringify({deviceId:'test', path:'/', signals:{}})})` returns ALLOW and a `Visit` row appears with your real IP.

### Done when
- [ ] Signed proxy requests work; unsigned ones get 401
- [ ] Real visitor IP confirmed (or fallback endpoint in place)
- [ ] `Visit` saved for each call; errors return ALLOW
- [ ] Unit tests green

---

## F03 · Store script (theme extension)

**Hours:** 3.5 · **Branch:** `feat/f03-store-script` · **Depends on:** F02

### Goal
A small script runs on every storefront page once the merchant enables the app embed. It calls the visitor check, shows the block screen when told to, and passes flags to the cart for checkout rules.

### Database
None.

### Backend
- **App embed:** `extensions/cs-storefront/blocks/sentinel.liquid`, `"target": "body"`. It loads `assets/sentinel.js` with `defer` and passes settings through `data-` attributes:
  - shop domain
  - proxy path
  - contact email (from a shop metafield written by the server on settings save)
- **Server:** on `updateSettings`, write `contactEmail` to a shop metafield `$app:cart-sentinel.public` so Liquid can read it.

### Frontend (storefront script `sentinel.js`)
1. **Device ID:** read the `_cs_did` cookie; if missing, create a random UUID and set it for 1 year (`SameSite=Lax`, `Secure`).
2. **Signals:** `collectSignals()` returns `{}` for now; filled in F04.
3. **Check:** `POST /apps/cs/check` with `{deviceId, path, signals}`. Timeout 2 s; on timeout or error do nothing (fail open).
4. **Act on the response:**
   - `BLOCK` → full-screen overlay (fixed, z-index max, scroll locked):
     *"Access blocked. We detected automated or unusual traffic. If you think this is a mistake, contact `<contactEmail>` and quote reference `<ref>`."*
   - `MONITOR` → `POST /cart/update.js` with attributes `{ "_cs_flags": "datacenter,tor", "_cs_did": "<deviceId>" }`.
   - `ALLOW` → set only `_cs_did` on the cart, once per session (remembered in `sessionStorage`) to avoid extra cart writes.
5. Runs once per page load. No framework, no external dependencies, ≤ 15 KB minified.

### Tests
- **Unit:** none (browser code). Keep the logic in small functions so F04 can unit-test detectors with jsdom.
- **Manual:**
  - Enable the app embed in the theme editor.
  - Open the store. The network tab shows `/apps/cs/check`, a `Visit` row appears, and the `_cs_did` cookie is set.
  - Temporarily force `BLOCK` with a dev-only flag → the overlay shows the contact email and reference.
  - Force `MONITOR` → `/cart.js` shows the `_cs_flags` attribute.
  - Lighthouse performance on a product page drops by ≤ 2 points with the embed on.

### Done when
- [ ] Script loads on all page types (home, collection, product, cart, search)
- [ ] Block overlay, monitor cart attribute and fail-open behaviour all work
- [ ] No visible page-speed impact

---

## F04 · Automation and extension checks

**Hours:** 3.5 · **Branch:** `feat/f04-browser-checks` · **Depends on:** F03

### Goal
The script detects automation tools and scraper extensions, and the server turns the results into `automation` and `extension` flags.

### Database
None (uses `Settings.extensionSigs` from F01).

### Backend
`app/bot/browser-signals.ts` turns raw signals into flags:
- **`automation`** if any **strong** signal is present, or **two or more weak** ones.
- **`extension`** if any **known** signature matched.
- Generic extension activity is logged only (flag `extension_generic`, never blocked). Password managers, Grammarly and similar inject into pages too.

### Frontend (storefront script detectors)

| Signal | Strength | Check |
|---|---|---|
| `webdriver` | strong | `navigator.webdriver === true` |
| `headless_ua` | strong | user agent contains `HeadlessChrome` |
| `automation_globals` | strong | any of `__playwright__binding__`, `__pwInitScripts`, `callPhantom`, `_phantom`, `__nightmare`, `domAutomation`, `domAutomationController` on `window`, or `$cdc_` / `$wdc_` keys on `document` (ChromeDriver) |
| `webdriver_tampered` | strong | the `navigator.webdriver` getter on `Navigator.prototype` isn't native code (someone patched it to hide) |
| `no_plugins` | weak | Chrome desktop user agent with `navigator.plugins.length === 0` |
| `no_languages` | weak | `navigator.languages` empty |
| `chrome_missing` | weak | Chrome user agent but `window.chrome` undefined |

**Extension scan:**
- **Known:** for each entry in the server-provided signature list (`[{id, selector?, global?}]`), check `document.querySelector(selector)` or the `window[global]` existence. Run once on load, and once more 3 s later through a `MutationObserver`, because extensions inject late.
- **Generic:** count elements whose `src` / `href` starts with `chrome-extension://` or `moz-extension://`.
- **Signature list:** the list ships **empty but editable**. Researching the real markers of PPSpy, Koala Inspector, ShopHunter etc. (install each on the dev store and inspect the DOM) is a task in this PR. Add what you find as defaults.

**Payload sent to `/apps/cs/check`:**

```json
{ "signals": { "automation": { "strong": ["webdriver"], "weak": ["no_plugins"] },
               "extensions": { "known": ["ppspy"], "generic": 2 } } }
```

### Tests
- **Unit (jsdom):** each detector returns the right signal with stubbed `navigator` / `window`. `browser-signals.ts`: 1 strong → automation; 1 weak → nothing; 2 weak → automation; known extension → extension; generic only → `extension_generic` only.
- **Playwright** (`tests/bots/automation.spec.ts`):
  - Plain Playwright Chromium visits the store → `Visit.flags` contains `automation`, response `BLOCK`.
  - Playwright with a stealth setup → record what's still detected (document the result; don't fail the PR if stealth gets through).
  - A normal Chrome visit has no flags.
- **Manual:** install one known scraper extension on your own browser → `extension` flag.

### Done when
- [ ] Default Playwright is blocked; normal Chrome is allowed
- [ ] At least one real spy extension signature added and detected
- [ ] Generic extensions never cause a block

> ⚠️ Until F05 adds the good-bot allowlist, keep `automation` on **MONITOR** on any store that real search engines can reach. Googlebot renders pages with headless Chrome.

---

## F05 · IP intelligence

**Hours:** 5.0 · **Branch:** `feat/f05-ip-intel` · **Depends on:** F02

### Goal
The server classifies every visitor IP (data-centre, Tor, blocked network, blocked country, too fast, verified good bot) and adds the matching flags.

### Database
None. IP data lives in files under `DATA_DIR` (gitignored).

### Backend

| Module | Job |
|---|---|
| `app/ip/geo.ts` | Opens `GeoLite2-ASN.mmdb` and `GeoLite2-Country.mmdb` with the `maxmind` npm package. `lookup(ip) → { asn, asnOrg, country }`. |
| `app/ip/ranges.ts` | Loads CIDR lists into a fast prefix lookup: AWS `ip-ranges.json`, Google Cloud `cloud.json`, Azure service tags, DigitalOcean, Oracle Cloud; plus the Tor bulk exit list. `isCloud(ip)`, `isTor(ip)`. |
| `app/ip/hosting-asns.ts` | Hard-coded hosting ASNs (Amazon 16509 & 14618, Google 15169 & 396982, Microsoft 8075, DigitalOcean 14061, Hetzner 24940, OVH 16276, Linode/Akamai 63949, Vultr 20473, Oracle 31898, Alibaba 45102, Contabo 51167, …). `isHostingAsn(asn)`. |
| `app/ip/good-bots.ts` | If the user agent claims Googlebot / bingbot / Applebot / DuckDuckBot / facebookexternalhit / Pinterestbot: reverse-DNS the IP, check the host ends with the bot's official domain (`googlebot.com`, `google.com`, `search.msn.com`, `applebot.apple.com`, …), forward-resolve to confirm the same IP. Facebook is verified by ASN 32934 instead. Results cached in memory for 24 h. `isGoodBot(ip, ua)`. |
| `app/ip/speed.ts` | In-memory sliding 60-second window. Count per IP and per subnet (/24 IPv4, /48 IPv6). Flag `speed` when IP count > `speedLimitPerMin` or subnet count > 5× that. Note: one app instance only in v1. |
| `app/jobs/refresh-ip-data.ts` | Downloads GeoLite2 (daily), cloud lists (daily) and the Tor list (hourly) to `DATA_DIR`, then reloads them in memory. Keeps the last good files if a download fails. Runs once on boot if files are missing. |

**Flags added in `proxy.check`:**
- `good_bot` — overrides everything else.
- `datacenter` — cloud range or hosting ASN.
- `tor`
- `asn_blocked` — ASN is in `blockedAsns`.
- `country_blocked` — country is in `blockedCountries`.
- `speed`

**Dev override:** when `CS_DEV_IP_OVERRIDE=true`, a request header `x-cs-test-ip` replaces the visitor IP. This lets you test "AWS IP", "Tor IP" and "home IP" without a VPN. The header must be ignored in production.

### Frontend
None (settings UI in F06).

### Tests
- **Unit:**
  - `ranges` — a known AWS IP is cloud, `8.8.8.8` is not, a sample Tor IP is Tor (use fixture files, not live downloads).
  - `speed` — the 61st call within a minute flags; calls spread over 2 minutes don't.
  - `good-bots` — mock DNS: a real Googlebot host passes; a fake host fails; a forward mismatch fails.
  - `geo` — reads a test `.mmdb` (MaxMind publishes test databases).
- **Playwright:** with the dev override, send `x-cs-test-ip` = an AWS IP → `datacenter` flag; a Tor IP → `tor` flag and `BLOCK`.
- **Manual:** run the refresh job and check the files in `data/`. Add your country to `blockedCountries` in Prisma Studio → you get blocked; remove it → allowed.

### Done when
- [ ] All six flags produced correctly
- [ ] Verified good bots always allowed; fake "Googlebot" user agents are not
- [ ] IP data refreshes on schedule and survives a failed download
- [ ] Dev override works locally and is ignored in production

---

## F06 · Bot Protection page

**Hours:** 4.0 · **Branch:** `feat/f06-bot-protection-page` · **Depends on:** F02–F05

### Goal
The merchant controls every bot check, manages blocked lists and the allowlist, and can find and unblock blocked visitors.

### Database

```prisma
enum AllowType { IP DEVICE EMAIL CUSTOMER }

model AllowlistEntry {
  id        String    @id @default(cuid())
  shop      String
  type      AllowType
  value     String              // IP, device ID, lower-cased email, or Shopify customer GID
  note      String?
  createdAt DateTime  @default(now())

  @@unique([shop, type, value])
}
```

### Backend
- **`app/services/allowlist.server.ts`:** `add`, `remove`, `list`, `isAllowlisted(shop, {ip, deviceId, email, customerId})`.
- **`proxy.check`:** if IP or device is allowlisted, add the `allowlisted` flag, which means `ALLOW`.
- **Customer entries:** adding a `CUSTOMER` entry also adds the Shopify tag `cs-trusted` to that customer (`tagsAdd`); removing it removes the tag. The checkout function (F09) reads this tag.
- **Unblock action:** given a Visit `ref`, add that visit's device ID and IP to the allowlist.

### Frontend — `app.bot-protection.tsx`

| Section | Contents |
|---|---|
| **Checks** | One row per check (Automation tools, Scraper extensions, Data-centre / VPN, Tor, Too many pages, Blocked networks, Blocked countries), each with an Off / Monitor / Block selector and a one-line explanation. A warning appears next to Data-centre when set to Block: *"Some real shoppers use VPNs."* |
| **Lists** | Blocked ASNs (numbers, one per line), Blocked countries (multi-select), Page speed limit (number), Contact email for the block screen. |
| **Allowlist** | Table of entries with type, value, note, Remove. "Add" form: type + value + note. |
| **Blocked visitors** | Last 7 days of `BLOCK` visits: time, reference, IP, country, network, reasons, page. Search by reference or IP. **Unblock** button. |

The page saves with one Save bar and shows success / error toasts.

### Tests
- **Unit:** `isAllowlisted` for each type; unblock adds both device and IP.
- **Manual:**
  - Block yourself with the dev IP override, find your reference on the page, click Unblock → next page load is allowed.
  - Set Tor to Off → Tor test IP allowed.
  - Add a customer to the allowlist → the customer shows the `cs-trusted` tag in Shopify admin.

### Done when
- [ ] Every setting saves and changes the next decision
- [ ] Allowlist works for IP and device on the storefront
- [ ] Unblock from the blocked-visitors list works end to end

---

## F07 · Rule engine core

**Hours:** 3.5 · **Branch:** `feat/f07-rule-engine` · **Depends on:** F01

### Goal
Rules can be saved, ordered and evaluated. The engine is pure, typed and fully unit-tested before any Shopify wiring.

**How rules run**
- There are two separate lists: **Checkout rules** and **Post-order rules**.
- Each list runs **top to bottom**. The **first enabled rule whose conditions match is applied, and every rule below it is skipped**.
- **Allow** is a real action. It does nothing to the order, but it stops lower rules from running. Merchants use it for exceptions, e.g. *"Email contains @mycompany.com → Allow"* placed at the top.
- A post-order rule in **test mode** that matches is logged, and evaluation carries on to the next rule. A test rule never acts and never stops the list.
- Disabled rules are skipped.
- **Missing data never matches.** If a field has no value (no billing phone, no IP), every condition on it is false, including "does not contain" and "none of".

### Database

```prisma
enum RuleStage { CHECKOUT ORDER }
enum MatchType { ALL ANY }

model Rule {
  id              String    @id @default(cuid())
  shop            String
  stage           RuleStage
  position        Int                         // 1 = top of the list
  name            String
  match           MatchType @default(ALL)
  conditions      Json                        // Condition[]
  actions         Json                        // CheckoutActions | OrderActions
  enabled         Boolean   @default(true)
  testMode        Boolean   @default(false)   // ORDER rules only
  timesTriggered  Int       @default(0)
  lastTriggeredAt DateTime?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([shop, stage, position])
}
```

Positions are always `1..n` with no gaps. Every move, create and delete rewrites positions inside one transaction.

### Backend

**`app/rules/types.ts`** — Zod schemas, shared by the server, the builder UI and the checkout function config:

```ts
type MatchOp =
  | "contains" | "not_contains" | "exact" | "not_exact" | "starts_with"
  | "any_of" | "none_of"
  | "gt" | "lt" | "eq" | "between"
  | "is_true" | "is_false"
  | "in_range";                              // IP CIDR

interface Condition {
  field: FieldKey;                           // Appendix A
  op: MatchOp;
  values: string[];                          // one or more; numbers as strings; between = [min, max]
  window?: { amount: number; unit: "hours" | "days" };   // velocity + cross-customer
}

type CheckoutActions = {
  main: { type: "allow" } | { type: "hide_payment"; methods: string[] };
};

type OrderActions = {
  main:
    | { type: "allow" }
    | { type: "hold" }
    | { type: "cancel"; refund: boolean; restock: boolean; notifyCustomer: boolean };
  tag?:   { target: "order" | "customer" | "both"; tags: string[] };
  email?: boolean;                           // email notification to the merchant
};
```

**Matching rules for values**
- **Text:** case-insensitive and trimmed. Phones are compared on digits only. With several values, the condition is true if **any** value matches (e.g. Contains `mailinator, guerrillamail`).
- **Negative text ops** (`not_contains`, `not_exact`) are true only if **no** value matches.

**`app/rules/catalog.ts`** — one entry per field:
- `key`, `label` (shown as "Filter Type"), `group`
- `valueType`: text, zip, country, province, provinceCode, number, money, bool, choice, ip, window-count
- allowed `ops`, `stages`, and `options` for choice fields

See [Appendix A](#appendix-a--condition-catalog).

**`app/rules/evaluate.ts`** — pure functions:
- `evaluateRule(rule, ctx) → { matched, reasons[] }`.
  - ALL = every condition true; ANY = at least one.
  - Reasons are plain English, e.g. *"Order Total 450.00 is greater than 300"*, *"Email domain mailinator.com is disposable"*.
- `runList(rules, ctx) → { applied?: {rule, reasons}, testMatches: {rule, reasons}[] }`:
  - walks rules by `position`, skips disabled ones
  - logs test-mode matches and carries on
  - stops at the first non-test match

**`app/rules/rules.server.ts`** — data access:
- `list(shop, stage, {search, page})` — 20 per page, by position.
- `create` — adds the rule at the **bottom** (position = n + 1).
- `update`, `remove` (closes the gap), `setEnabled`.
- `moveUp` / `moveDown` — swap with the neighbour in a transaction.

**Validation (Zod)**
- Name 1–80 characters; at least one condition.
- Field allowed for the stage; op allowed for the field.
- Values valid for the type: number parses; `between` has min < max; CIDR is valid; country codes are ISO-2; a window is present for window fields.
- Actions match the stage; `testMode` only on ORDER rules; `hide_payment` has at least one method.

### Frontend
None in this PR (UI in F11). Add a dev seed script `npm run seed:rules` that creates sample rules in both lists for testing F08–F10.

### Tests
Unit tests are table-driven, with fixtures in `tests/fixtures/rules/*.json`:
- Every op × every value type, including multi-value text and `between` edges.
- ALL vs ANY.
- Missing field → false for every op.
- `runList`:
  - first match wins
  - disabled skipped
  - Allow stops the list
  - test match logged and evaluation continues
  - test match followed by a live match → both reported, only the live one applied
  - empty list → nothing
- Positions stay `1..n` after create, delete and moves.
- Validation rejects:
  - an order-only field on a checkout rule
  - `hold` on a checkout rule
  - `testMode` on a checkout rule
  - `between` with min ≥ max
  - a bad CIDR

The same fixtures are reused by the checkout function tests in F09, so both evaluators agree.

### Done when
- [ ] `evaluateRule` and `runList` fully covered by unit tests
- [ ] Invalid rules can't be saved
- [ ] Reordering never leaves gaps or duplicates
- [ ] Seed script creates working sample rules in both lists

---

## F08 · Rule conditions and order intake

**Hours:** 6.5 · **Branch:** `feat/f08-conditions-intake` · **Depends on:** F05, F07

### Goal
Every new order is received, turned into a context with all condition fields, evaluated against ORDER rules and saved with the matched rules and reasons. **No actions yet** — this PR behaves as if every rule were in Test mode.

### Database

```prisma
enum CheckStatus { PASSED REVIEW CANCELLED TAGGED APPROVED FRAUD ERROR }

model OrderCheck {
  id           String      @id @default(cuid())
  shop         String
  orderId      String                      // Shopify GID
  orderName    String                      // "#1042"
  ip           String?
  emailHash    String?
  phoneHash    String?
  total        Decimal     @db.Decimal(12, 2)
  currency     String
  gateway      String?                     // e.g. "Cash on Delivery (COD)"
  customerKey  String?                     // customer GID, else email hash (for cross-customer counts)
  nameHash     String?
  shipAddrHash String?
  appliedRule  Json?                       // {ruleId, name, reasons[]} — the first live rule that matched
  testMatches  Json        @default("[]")  // test-mode rules that matched: [{ruleId, name, reasons[]}]
  actionsTaken Json        @default("[]")  // F10: [{type, ok, error?, at}]
  status       CheckStatus @default(PASSED)
  createdAt    DateTime    @default(now())

  @@unique([shop, orderId])
  @@index([shop, status, createdAt])
  @@index([shop, ip, createdAt])
  @@index([shop, emailHash, createdAt])
  @@index([shop, phoneHash, createdAt])
  @@index([shop, nameHash, createdAt])
  @@index([shop, shipAddrHash, createdAt])
}

enum FraudType { EMAIL PHONE ADDRESS }

model FraudEntry {
  id        String    @id @default(cuid())
  shop      String
  type      FraudType
  valueHash String
  orderId   String?                // order that confirmed the fraud
  createdAt DateTime  @default(now())

  @@unique([shop, type, valueHash])
}
```

(`FraudEntry` rows are written by "Mark as fraud" in F12; this PR only reads them.)

### Backend

**Webhook** `webhooks.orders.create.tsx` (subscribed in `shopify.app.toml`):
1. Verify the webhook (template helper) and return **200 immediately**.
2. Process in the background. Idempotency: skip if `OrderCheck(shop, orderId)` exists, since Shopify retries.

**`app/orders/fetch.ts`** — one GraphQL query for the order:
- email, phone
- billing and shipping address
- client IP
- custom attributes (`_cs_flags`, `_cs_did`)
- payment gateway names
- total + currency
- line items (quantity, product ID)
- customer (ID, number of orders, tags)
- Shopify risk assessments (highest risk level)
- full billing and shipping addresses including names, company and phone
- Shopify Protect status (where the store has it)
- disputes (chargebacks) on the customer's previous orders — needs the disputes read scope; empty for stores without Shopify Payments

Check the field names against the pinned API version.

**`app/services/hash.ts`:**
- `normalizeEmail` — lower-case, trim.
- `normalizePhone` — digits only, with country code via `libphonenumber-js` when possible.
- `normalizeAddress` — lower-case, strip punctuation, collapse spaces, common abbreviations (street→st, road→rd, apartment / apt / # → apt, house→h), then `address1 + address2 + zip + country`.
- `hash(shop, value) = HMAC-SHA256(HASH_SECRET, shop + ":" + value)`.

**`app/orders/context.ts`** — `buildOrderContext(order, shop) → OrderContext` with every ORDER field in [Appendix A](#appendix-a--condition-catalog):
- Billing and shipping fields: copied straight from the order (all 12 of each in Appendix A).
- Disposable email: bundled JSON list from the open-source `disposable-email-domains` project, loaded once.
- Chargebacks: count of disputes on the customer's earlier orders.
- Cross-customer: count of **different** `customerKey`s with the same `nameHash` / `phoneHash` / `shipAddrHash` inside the condition's window.
- IP fields: reuse F05 `geo` / `ranges` / `hosting-asns` on the order's client IP.
- Velocity: count `OrderCheck` rows in the same shop with the same IP / email hash / phone hash within `param` hours (excluding this order).
- Past fraud: look up `FraudEntry` for the email, phone, shipping address and billing address hashes. Also exposes **which** one matched for the reason text.
- Allowlist: if email or customer is allowlisted, skip evaluation entirely (status `PASSED`, reason "Allowlisted").

**`app/orders/intake.ts`** — fetch → context → `runList(post-order rules)` (F07) → save `OrderCheck` with `appliedRule` and `testMatches` → increment `timesTriggered` / `lastTriggeredAt` on the applied rule and on test matches. Any error → `OrderCheck.status = ERROR` with the message; the order is left alone.

### Frontend
None (Orders page in F12). Temporary: the app home lists the last 10 `OrderCheck` rows with matched rule names.

### Tests
- **Unit:**
  - `normalize*` + `hash` (same address in two spellings → same hash).
  - `buildOrderContext` from fixture order JSON for each field: mismatch booleans, disposable email, IP country mismatch, data-centre IP, risk level, velocity counts, past-fraud match.
  - Intake is idempotent (second webhook for the same order does nothing).
- **Manual (with seed rules):**
  - Place a test order with a `mailinator.com` email → `OrderCheck` shows "Disposable email" matched.
  - Place 3 orders from the same browser → velocity rule matches on the third.
  - Insert a `FraudEntry` for your test email hash → the next order matches "past fraud".

### Done when
- [ ] Every order creates exactly one `OrderCheck`
- [ ] All ORDER conditions in Appendix A produce correct values
- [ ] Webhook always answers 200 within a second

---

## F09 · Checkout function

**Hours:** 5.5 · **Branch:** `feat/f09-checkout-function` · **Depends on:** F07

### Goal
Enabled checkout rules hide COD or other payment methods during checkout. The first matching rule in the Checkout list decides.

> **Confirm first:** which Shopify plans let an App Store app's payment customization run, and that hiding works with the store's checkout. Write the answer in the PR.

### Database
None. The rule config lives in a metafield on the payment customization object.

### Backend

**Function extension** `extensions/cs-payment-rules` (JavaScript, target `cart.payment-methods.transform.run`).

**Input query** (field names to confirm against the current API; use **input variables** from the config metafield so `inAnyCollection` and `hasAnyTag` get the merchant's IDs and tags):

```graphql
query RunInput($collectionIds: [ID!], $customerTags: [String!]) {
  cart {
    cost { totalAmount { amount } }
    lines {
      quantity
      merchandise { ... on ProductVariant { product { id inAnyCollection(ids: $collectionIds) } } }
    }
    deliveryGroups { deliveryAddress { countryCode zip city address1 } }
    buyerIdentity { email customer { numberOfOrders hasAnyTag(tags: $customerTags) } }
    flags: attribute(key: "_cs_flags") { value }
  }
  paymentMethods { id name }
  paymentCustomization {
    metafield(namespace: "$app:cart-sentinel", key: "config") { jsonValue }
  }
}
```

- **Run logic:**
  - If the customer has the `cs-trusted` tag, hide nothing.
  - Otherwise walk the config rules **in list order** with a **small evaluator inside the function** that supports only checkout fields.
  - The **first** matching rule decides:
    - `allow` → hide nothing
    - `hide` → hide every payment method whose name contains one of its `methods` (case-insensitive)
  - Later rules are ignored.
  - Never hide **all** methods: if everything would be hidden, hide nothing and stop. A store with no way to pay is worse than a missed rule.
- **Config shape** (kept small; max 25 checkout rules):

  ```json
  { "v": 1, "trustedTag": "cs-trusted",
    "rules": [{ "id": "r1", "match": "ALL",
                "conditions": [{ "field": "order.total", "op": "gt", "values": ["300"] }],
                "action": "hide", "methods": ["cash on delivery"] }] }
  ```

**Server** `app/services/payment-customization.server.ts`:
- `ensureCustomization(shop)` — find our payment customization or create it (`paymentCustomizationCreate` with the function ID, enabled).
- `syncCheckoutRules(shop)` — build the config from **enabled** CHECKOUT rules **in list order** and write it with `metafieldsSet`. Called after every rule create / update / delete / enable / disable / move.
- `getStatus(shop)` — whether the customization exists and is enabled.

### Frontend
- Status card on the app home: **"Checkout protection: Active / Not active"** with a "Fix" button that runs `ensureCustomization`.

### Tests
- **Function tests:** run with `shopify app function run` against input fixtures:
  - over the cart limit → COD hidden
  - under the limit → nothing hidden
  - trusted customer → nothing hidden
  - visitor flag `datacenter` → COD hidden
  - a rule that would hide all methods → nothing hidden
  - an Allow rule above a hide rule, both matching → nothing hidden (first match wins)

  Reuse the F07 fixture conditions so both evaluators agree.
- **Unit:** `syncCheckoutRules` builds the expected config in list order; disabled rules are excluded.
- **Manual (dev store):** seed "Order Total greater than 300 → Hide payment methods: Cash on Delivery".
  - Cart of 250 → COD visible.
  - Cart of 350 → COD gone, card still there.
  - Disable the rule → COD back within a minute.

### Done when
- [ ] COD disappears only when the first matching enabled rule hides it
- [ ] Trusted customers never lose payment methods
- [ ] Config re-syncs on every rule change

---

## F10 · Order actions

**Hours:** 5.0 · **Branch:** `feat/f10-order-actions` · **Depends on:** F08

### Goal
The first matching post-order rule now acts: hold fulfillment, cancel or allow, plus tag and email notification.

### Database
None (uses `OrderCheck.actionsTaken` and `status`).

### Backend

**`app/orders/actions.ts`** — after intake, take the **applied rule** (the first enabled, non-test rule that matched). Test matches never act. Run its main action first, then Tag, then Email notification:

| Action | Shopify call | Result |
|---|---|---|
| **Hold fulfillment** (main) | `fulfillmentOrderHold` on each open fulfillment order (`reason: OTHER`, notes: rule name) | status `REVIEW`; tag order `cs-review` |
| **Cancel** (main) | `orderCancel` with `reason: FRAUD` and the rule's `refund`, `restock`, `notifyCustomer` | status `CANCELLED`; tag order `cs-cancelled` |
| **Allow** (main) | nothing | status `PASSED`; rule name recorded |
| **Tag** (information) | `tagsAdd` on the order and/or customer (customer only if the order has one) | status `TAGGED` if the main action was Allow |
| **Email notification** (information) | Email to `Settings.alertEmail` (fallback: shop owner email): order name, total, rule name, reasons, link to the order in our app | — |

- Each action records `{type, ok, error?, at}` in `actionsTaken`. One failing action doesn't stop the others.
- Retries: Shopify throttling errors retry up to 3 times with backoff. Other errors are recorded and shown on the Orders page.
- `app/services/email.server.ts`: Nodemailer with SMTP env vars; plain-text + simple HTML template.

### Frontend
None (Orders page in F12). The temporary home list now shows status and actions.

### Tests
- **Unit:** run actions with a mocked GraphQL client:
  - only the applied rule acts, never a rule below it
  - Allow with no Tag / Email makes no Shopify calls
  - one failing call recorded while the others still run
  - test-mode matches trigger nothing
- **Manual (dev store with seed rules), one test order per action:**
  - Review → the order shows "On hold" in Shopify and can't be fulfilled.
  - Cancel with refund + restock + email → order cancelled, refunded, stock back, email visible in Mailpit (with notify on, Shopify sends its own cancellation email).
  - Tag → tags visible on the order and customer.
  - Alert → email arrives in Mailpit.

### Done when
- [ ] Each of the four actions works on the dev store
- [ ] Only the first matching rule acts
- [ ] Failures visible in `actionsTaken`, never crash intake

---

## F11 · Rules page and rule builder

**Hours:** 7.0 · **Branch:** `feat/f11-rules-ui` · **Depends on:** F07–F10

### Goal
Merchants manage two ordered rule lists and build rules with a form, never JSON. The layout follows the agreed reference screens: a rules table with reorder arrows, and an Add Rule page with three cards (Rule Settings, Conditions, Actions).

### Database
None (uses `Rule` from F07).

### Backend
- Loaders and actions call `rules.server.ts` (F07): list, create, update, delete, enable/disable, move up/down.
- Every change to a **checkout** rule calls `syncCheckoutRules` (F09).
- **Payment method names for the "Hide payment methods" picker:** suggest the names seen on past orders (`OrderCheck.gateway`) plus "Cash on Delivery". Also allow free text, because there may be no simple API for an app to list a store's checkout payment methods (confirm in this PR).
- **Country and province lists:** a static ISO country list plus Shopify's country/province data for province pickers.

### Frontend

#### Rules list — `app.rules._index.tsx`

```
Rules                                                              [ Add rule ]
┌─────────────────────────────────────────────────────────────────────────────┐
│ Rules match in the order they are listed below. The first rule that matches │
│ will be applied.                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────────────┐
│ [Post-order rules ⌄]  [ 🔍 Search rules by name...                        ] │
│ Rule              Match  Condition(s)            Action(s)          Created  Actions │
│ Disposable Email  all    [Check if Disp. Email]  [Cancel]           Oct 15   ▶ ✎ 🗑 ↓ │
│ Address mismatch  all    [Billing - Country]     [Hold fulfillment] Oct 15   ▶ ✎ 🗑 ↑ ↓ │
│ Shopify High Risk all    [Shopify Risk Level]    [Tag][Email][Hold] Oct 15   ▷ ✎ 🗑 ↑ ↓ │
│ ...                                                                          │
│                         |◀   ◀   Page 1 of 1   ▶   ▶|                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

- **Banner:** "Rules match in the order they are listed below. The first rule that matches will be applied."
- **List selector:** Post-order rules / Checkout rules. Kept in the URL (`?stage=order|checkout`) so a refresh and the back button keep it.
- **Search** by name: server-side, debounced, partial match.
- **Columns:**

| Column | Content |
|---|---|
| Rule | Name, cut to 2 lines with "…"; full name in a tooltip. A **Test** badge if in test mode. |
| Match Type | Badge `all` or `any`. |
| Condition(s) | One chip per condition showing the Filter Type label. Tooltip shows the full condition, e.g. "Order Total greater than 300". |
| Action(s) | Chips: the main action (Allow / Hold fulfillment / Cancel / Hide payment methods), plus Tag and Email notification if set. |
| Created | Date in the store's locale. |
| Actions | Icon buttons: **▶ Enable / disable** (solid = enabled, greyed = disabled; tooltip says which) · **✎ Edit** · **🗑 Delete** (red; confirmation modal "Delete rule '<name>'? This can't be undone.") · **↑ Move up** (hidden on the first rule) · **↓ Move down** (hidden on the last rule). Moving works across page boundaries. |

- **Pagination:** 20 rules per page with first / previous / "Page X of Y" / next / last.
- **Empty state** for each list: a short explanation and an "Add rule" button.
- **Toasts:** "Rule enabled", "Rule disabled", "Rule moved", "Rule deleted".

#### Add / edit rule — `app.rules.new.tsx` and `app.rules.$id.tsx`

- **Breadcrumb:** Rules / Add New Rule (or Rules / Edit Rule).
- **List:** taken from `?stage=` and shown as a badge ("Post-order rule" / "Checkout rule"). It can't be changed after creation.

**Card 1 — Rule Settings**
- **Rule Name** (required, max 80).
- **How should conditions combine?** (radio)
  - **Match only when all conditions are true (AND)** — "Safer default. Every condition must match before this rule runs." *(default)*
  - **Match when at least 1 condition is true (OR)** — "Use with care. One broad condition can trigger this rule even if every other condition fails."
- Post-order rules only: **☐ Test mode** — "Log matches on the Orders page without taking any action."

**Card 2 — Conditions** (one or more rows)

Each row has three inputs:
- **Filter Type:** a grouped select with the groups and labels from [Appendix A](#appendix-a--condition-catalog). It shows only fields available for this list.
- **Match Type** and **Value:** depend on the field type:

| Field type | Match Type options | Value input |
|---|---|---|
| Text (names, company, address lines, city, email, phone) | Contains · Exact match · Does not contain · Not exact match | Tag input: one or more values; true if any matches |
| Postal code | Exact match · Starts with · Not exact match | Tag input |
| Country | Any of · None of | Searchable multi-select ("Search Countries") |
| Province / State | Any of · None of | Searchable multi-select of provinces for the countries already chosen in this rule; free text otherwise |
| Province code | Any of · None of | Tag input (e.g. `NY`, `ON`) |
| Number / money (order total, quantities, order count, chargebacks) | Greater than · Less than · Equal to · Between | Number input; Between shows two. Money shows the store currency. |
| Yes / No check (disposable email, address mismatch, data-centre IP, past fraud…) | Is true · Is false | none |
| Choice (Shopify Risk Level, Protect eligibility, payment method, visitor flags) | Any of · None of | Multi-select of the fixed options |
| IP address | Exact match · Starts with · In range (CIDR) | Tag input, validated |
| Count in a time window (velocity, cross-customer) | Greater than | Number + "in the last [N] [hours / days]" |

- Changing the Filter Type resets that row's Match Type and Value.
- 🗑 removes a row; it's hidden when only one row is left.
- **+ Add condition** adds a row under the last one.

**Card 3 — Actions**

*Post-order rules*
- **Order Action** (radio, pick one): **Allow** · **Hold fulfillment** · **Cancel**.
  - Cancel shows: ☑ Refund payment · ☑ Restock items · ☑ Notify customer (all ticked by default).
- **Information** (optional, any combination):
  - ☐ **Tag** → apply to Order / Customer / Both, plus a tag input.
  - ☐ **Email notification** → "Send an email to `<alert email>`". If no alert email is set, a link to set one.
- **Info banner** when Allow is chosen with no Tag or Email:
  > **Only "allow" action selected.** This rule will still match according to the condition(s), but it will do nothing. It is purely informational and will prevent rules with lower priority from being executed.

*Checkout rules*
- **Checkout Action** (radio): **Allow** · **Hide payment methods** → multi-select of method names (default "Cash on Delivery") with free-text entry.
- Same Allow banner.

**Saving**
- Shopify's contextual save bar (Save / Discard).
- New rules go to the **bottom** of their list. The toast says *"Rule added at the bottom of the list. Move it up if it should run earlier."*
- Validation errors appear under each input, plus a summary banner at the top.
- Leaving with unsaved changes asks for confirmation.

**Starter templates** (optional; about 1h of this feature): an "Add from template" menu on the list opens the builder pre-filled. Template rules are saved **disabled**.

| Template | List | Conditions | Action |
|---|---|---|---|
| Hide COD for big orders | Checkout | Order Total greater than 300 | Hide "Cash on Delivery" |
| Hide COD for risky visitors | Checkout | Visitor flagged any of datacenter, tor | Hide "Cash on Delivery" |
| Hide COD for fraud customers | Checkout | Customer tag any of `cs-fraud` | Hide "Cash on Delivery" |
| Cancel known fraud | Post-order | Matches past fraud is true | Cancel (refund, restock) + tag `cs-fraud` + email |
| Disposable email | Post-order | Check if Disposable Email is true | Hold fulfillment + tag |
| Address mismatch | Post-order | Billing vs. Shipping Address Mismatch is true | Hold fulfillment + tag + email |
| Shopify high risk | Post-order | Shopify Risk Level any of High | Hold fulfillment + tag + email |
| Micro orders (card testing) | Post-order | Order Total less than 2 | Hold fulfillment + tag + email |
| New customer, big order | Post-order | Customer Order Count less than 1 AND Order Total greater than 500 | Hold fulfillment + tag + email |

### Tests
- **Unit:**
  - Form state ↔ rule JSON round-trip for every field type.
  - Match Type options per field type.
  - The Filter Type list is filtered by stage.
  - Between requires min < max.
  - The Allow banner shows only for Allow with no extras.
  - Every template passes validation.
- **Manual:**
  - Recreate the reference list (Disposable Email → Cancel, Address mismatch → Hold, Shopify High Risk → Tag + Email + Hold, …). Check chips, badges, dates, truncation and the disabled ▶ state.
  - Move rules up and down, including the first row, the last row and across pages. The order survives a refresh.
  - Disable a rule → an order that matched it now falls through to the next rule.
  - Put "Email contains `<your test email>` → Allow" at the top → your test orders are no longer held.
  - Delete with the confirmation modal.
  - Checkout list:
    - Create "Order Total greater than 300 → Hide payment methods: Cash on Delivery" → COD hidden at checkout.
    - Move an Allow rule above it → COD visible again.
  - Search by partial name; pagination at 21+ rules.

### Done when
- [ ] Rules list matches the reference layout for both lists
- [ ] Every field in Appendix A can be added with the right Match Types and value input
- [ ] Enable/disable, edit, delete and move up/down all persist
- [ ] Checkout rule changes reach the checkout function

---

## F12 · Orders page

**Hours:** 4.5 · **Branch:** `feat/f12-orders-page` · **Depends on:** F10

### Goal
Staff see every flagged order, why it was flagged, and can approve, cancel or mark it as fraud.

### Database

```prisma
enum StaffActionType { APPROVE CANCEL MARK_FRAUD REMOVE_FRAUD UNBLOCK }

model StaffAction {
  id           String          @id @default(cuid())
  shop         String
  orderCheckId String?
  action       StaffActionType
  staffUserId  String?                     // from the admin session token, if available
  note         String?
  createdAt    DateTime        @default(now())

  @@index([shop, createdAt])
}
```

### Backend
- **List loader:**
  - Tabs: In review (`REVIEW`), Cancelled (`CANCELLED`), Tagged (`TAGGED`), All flagged (any with an applied rule or a test match), Errors (`ERROR`).
  - Paging (25 per page), filter by date range, search by order name.
- **Detail loader:** the `OrderCheck` plus live order details fetched from Shopify (customer name, email, phone, addresses, items). Contact details are fetched live, never stored in plain text.
- **Actions:**
  - **Approve** — `fulfillmentOrderReleaseHold` on held fulfillment orders, remove `cs-review` tag, status `APPROVED`. Optional "Also trust this customer" adds a `CUSTOMER` allowlist entry (F06).
  - **Cancel** — same as the F10 cancel, with Refund / Restock / Email customer options in a dialog.
  - **Mark as fraud** — dialog with checkboxes: add email / phone / shipping address / billing address to the fraud list (all ticked by default), and "Also cancel the order" (ticked if not already cancelled). Writes `FraudEntry` rows (hashed), tags the customer and order `cs-fraud`, status `FRAUD`.
  - **Remove from fraud list** — on the Fraud list tab.
  - Every action writes a `StaffAction` row.
- **Fraud list tab:** shows entries with type, order that created them and date. The value is a hash, so show *"Email from order #1042"* rather than the email.

### Frontend
- **`app.orders._index.tsx`:** tabs + table (order, customer name (live), total, rule applied, status badge, date). Clicking a row opens the detail page.
- **`app.orders.$id.tsx`:**
  - Header: order name, status badge, total, "Open in Shopify" link.
  - Rule card: the rule that was applied with its reasons, plus any test-mode rules that matched.
  - Customer card: name, email, phone (clickable `tel:`), addresses.
  - Actions taken card: timeline of automatic and staff actions, including errors.
  - Buttons: Approve, Cancel, Mark as fraud (shown according to status).

### Tests
- **Unit:** Mark as fraud writes the correct hashes and skips missing fields; Approve on a non-held order is a no-op with a message.
- **Manual — the full story:**
  1. A test order triggers the review rule → appears under In review.
  2. Open it → the reasons and phone are shown.
  3. Mark as fraud (cancel ticked) → order cancelled, customer tagged `cs-fraud`.
  4. Place a new order with the same email → F08 matches "past fraud" and the "Cancel known fraud" rule cancels it.
  5. At checkout, the fraud-tagged customer (logged in) doesn't see COD.

### Done when
- [ ] All tabs list the right orders
- [ ] Approve, Cancel and Mark as fraud work and are logged
- [ ] The full story above passes

---

## F13 · Dashboard

**Hours:** 2.5 · **Branch:** `feat/f13-dashboard` · **Depends on:** F02, F08

### Goal
One page shows what the app has done and that it's working.

### Database
None. Reads `Visit`, `OrderCheck`, `Rule`. Uses the indexes from F02 and F08.

### Backend
Loader with a range parameter (`today`, `7d`, `30d`). Queries:
- Visitors blocked, visitors flagged (monitor) — counts on `Visit`.
- Orders in review, cancelled, tagged — counts on `OrderCheck` by status.
- **Money saved** — sum of `total` for `CANCELLED` + `FRAUD` checks, grouped by currency.
- Top 5 block reasons — unnest `Visit.flags` where action is `BLOCK`.
- Top 5 rules — from `OrderCheck.appliedRule`.
- Setup health:
  - app embed enabled (the theme app embed setting via the Admin API, or "a visit was seen in the last 24 h")
  - checkout protection active (F09 `getStatus`)
  - IP data age (F05)

### Frontend — `app._index.tsx`
- **Setup checklist** (shown until complete):
  - Enable the app embed (deep link to the theme editor)
  - Checkout protection active
  - Create your first rule
- Range selector.
- Stat tiles: Blocked visitors · Flagged visitors · In review · Cancelled · Money saved.
- Two short lists: Top block reasons, Top rules (each linking to the filtered page).
- Empty state for new installs: *"Nothing blocked yet. That's normal for a new store."*

### Tests
- **Unit:** each query against seeded data gives the expected numbers; currencies aren't mixed.
- **Manual:** after the F12 story, the dashboard numbers match the Orders page and Bot Protection lists.

### Done when
- [ ] Numbers match the underlying pages for each range
- [ ] Setup checklist reflects real state

---

## F14 · Privacy and uninstall

**Hours:** 2.0 · **Branch:** `feat/f14-privacy` · **Depends on:** all tables

### Goal
The app meets Shopify's privacy requirements and doesn't keep data longer than needed.

### Database
None (deletes rows).

### Backend
- **Compliance webhooks** (declared in `shopify.app.toml` as compliance topics):
  - `customers/data_request` — collect our data for the listed orders (`OrderCheck` rows) and email the summary to the shop's alert email. Log the request.
  - `customers/redact` — delete `OrderCheck` rows for the listed orders. `FraudEntry` rows contain only hashes; keeping them for fraud prevention is a **legal decision** (Appendix D). Until decided, delete entries whose `orderId` is in the list.
  - `shop/redact` — delete every row for the shop in every table.
- **`app/uninstalled`:** delete sessions, set `Shop.uninstalledAt`. Data is fully wiped when `shop/redact` arrives (Shopify sends it 48 hours after uninstall).
- **Retention job** (daily): delete `Visit` older than 30 days, `OrderCheck` older than 365 days, `StaffAction` older than 365 days.

### Frontend
- A short **"Data & privacy"** section at the bottom of the Bot Protection page saying what's stored, for how long, and that values are hashed. The App Store listing also needs a privacy policy URL (written outside the code).

### Tests
- **Unit:** each handler deletes exactly the right rows for the right shop and nothing from other shops; the retention job deletes only old rows.
- **Manual:** trigger the compliance webhooks with `shopify app webhook trigger` and check the rows. Uninstall and reinstall the app on the dev store.

### Done when
- [ ] All three compliance webhooks handled and tested
- [ ] Uninstall marks the shop; `shop/redact` wipes it
- [ ] Retention job runs daily

---

## F15 · Pricing plan

**Hours:** 2.0 · **Branch:** `feat/f15-pricing` · **Depends on:** F08

### Goal
Merchants pick a paid plan; usage limits are enforced fairly.

> **Decide before building:** plan names, prices and limits. For reference, Fraud Falcon charges $29.99 for 500 orders/month, $49.99 for 1,000 and $149.99 for 5,000 (Appendix D).

### Database
Uses `Shop.plan` (F01). Add a field only if needed:

```prisma
// add to Shop
planOrderLimit Int?    // null = unlimited
```

### Backend
- Use Shopify **Managed Pricing** (plans defined in the Partner Dashboard; Shopify hosts the plan page) if available for the app. Otherwise use the Billing API's `billing.require` from the template. Confirm which one applies.
- On each request to an admin page, read the active subscription (cached for 10 minutes) and update `Shop.plan` / `planOrderLimit`.
- **Limit:** count `OrderCheck` rows this calendar month. When over the limit:
  - Bot Block and checkout rules keep working.
  - After-order rules switch to log-only.
  - The merchant sees a banner. The rules never silently stop.
- Development stores: free, no limit.

### Frontend
- Plan banner on the Dashboard: current plan, orders used this month / limit, "Change plan" button.
- Over-limit banner on every page.

### Tests
- **Unit:** the limit check at, below and above the limit; dev stores are never limited.
- **Manual:** select a test plan on the dev store; lower the limit in the database → the banner shows and after-order rules log without acting.

### Done when
- [ ] Plan selection works
- [ ] Limits enforced as specified, with a visible banner

---

## F16 · Go live

**Hours:** 2.0 · **Branch:** `feat/f16-go-live` · **Depends on:** all

### Goal
The app runs on hosted infrastructure and the full story works against it.

### Database
Hosted PostgreSQL. Migrations run on every deploy (`prisma migrate deploy`).

### Backend
- **Hosting:** one app instance (Fly.io, Render or Railway; pick one) + managed Postgres. Single instance because the speed limiter and good-bot cache are in memory.
- **Persistent volume** for `DATA_DIR` (IP files), or download on boot.
- **Env vars** from §0.6 set on the host; `CS_DEV_IP_OVERRIDE` unset.
- **Cron jobs** (in-process): IP data refresh (daily + Tor hourly), retention (daily).
- **`shopify app deploy`:** pushes the app config, theme extension and checkout function.
- **Production URLs:** `SHOPIFY_APP_URL`, app proxy URL and webhook URLs all point at the host.
- **Real SMTP provider** for alert emails.
- **Error logging:** host logs at minimum. An error tracker is optional.

### Frontend
None.

### Tests — smoke checklist on the hosted app
- [ ] Install on a fresh dev store; settings created
- [ ] App embed on → visits logged with real IPs
- [ ] Playwright bot run against the live store → blocked
- [ ] Checkout rule hides COD
- [ ] Post-order rules: hold, cancel, allow, tag and email notification each work
- [ ] The F12 full story passes
- [ ] Uninstall → sessions removed; compliance webhooks reach the host

### Done when
- [ ] Every smoke test passes on the hosted app
- [ ] Protected customer data request approved (needed before public listing)

---

## Appendix A — Condition catalog

Groups and labels are what the merchant sees in the **Filter Type** dropdown.
- ✅ = available, — = not available.
- **Checkout** fields come from the checkout function input. Customer fields there exist only for logged-in shoppers, and email only once entered.
- Confirm in F09 exactly which delivery address fields the function input exposes. Any that aren't there become post-order only.

| Group | Filter Type (label) | Key | Type | Checkout | Post-order |
|---|---|---|---|---|---|
| **Billing Address** | Billing Address - City | `billing.city` | text | — | ✅ |
| | Billing Address - Country | `billing.country` | country | — | ✅ |
| | Billing Address - Province/State | `billing.province` | province | — | ✅ |
| | Billing Address - Postal Code | `billing.zip` | zip | — | ✅ |
| | Billing Address - Address Line 1 | `billing.address1` | text | — | ✅ |
| | Billing Address - Address Line 2 | `billing.address2` | text | — | ✅ |
| | Billing Address - Company | `billing.company` | text | — | ✅ |
| | Billing Address - First Name | `billing.first_name` | text | — | ✅ |
| | Billing Address - Last Name | `billing.last_name` | text | — | ✅ |
| | Billing Address - Full Name | `billing.name` | text | — | ✅ |
| | Billing Address - Phone | `billing.phone` | text | — | ✅ |
| | Billing Address - Province Code | `billing.province_code` | provinceCode | — | ✅ |
| **Shipping Address** | Shipping Address - City | `shipping.city` | text | ✅ | ✅ |
| | Shipping Address - Country | `shipping.country` | country | ✅ | ✅ |
| | Shipping Address - Province/State | `shipping.province` | province | — | ✅ |
| | Shipping Address - Postal Code | `shipping.zip` | zip | ✅ | ✅ |
| | Shipping Address - Address Line 1 | `shipping.address1` | text | ✅ | ✅ |
| | Shipping Address - Address Line 2 | `shipping.address2` | text | ✅ | ✅ |
| | Shipping Address - Company | `shipping.company` | text | ✅ | ✅ |
| | Shipping Address - First Name | `shipping.first_name` | text | ✅ | ✅ |
| | Shipping Address - Last Name | `shipping.last_name` | text | ✅ | ✅ |
| | Shipping Address - Full Name | `shipping.name` | text | ✅ | ✅ |
| | Shipping Address - Phone | `shipping.phone` | text | ✅ | ✅ |
| | Shipping Address - Province Code | `shipping.province_code` | provinceCode | ✅ | ✅ |
| **Browser Detail** | IP Address | `ip.address` | ip | — | ✅ |
| | IP Country | `ip.country` | country | — | ✅ |
| | IP is Data-centre / VPN | `ip.datacenter` | bool | — | ✅ |
| | IP is Tor | `ip.tor` | bool | — | ✅ |
| | IP Country vs. Billing Country Mismatch | `ip.country_mismatch` | bool | — | ✅ |
| | Visitor Flagged by Bot Block | `visitor.flags` | choice: automation, extension, datacenter, tor, asn_blocked, country_blocked, speed | ✅ | ✅ |
| **Cross-Customer Detection** | Customer Name Used by Other Customers | `cross.name` | window-count | — | ✅ |
| | Phone Used by Other Customers | `cross.phone` | window-count | — | ✅ |
| | Shipping Address Used by Other Customers | `cross.address` | window-count | — | ✅ |
| **Customer** | Email | `customer.email` | text | ✅ | ✅ |
| | Phone Number | `customer.phone` | text | — | ✅ |
| | Customer Name | `customer.name` | text | — | ✅ |
| | Customer Order Count | `customer.order_count` | number | ✅ | ✅ |
| | Customer Chargebacks | `customer.chargebacks` | number | — | ✅ |
| | Check if Disposable Email | `customer.disposable_email` | bool | — | ✅ |
| | Customer Tag | `customer.tags` | choice (free tags) | ✅ | ✅ |
| **Order** | Order Total | `order.total` | money | ✅ | ✅ |
| | Total Item Quantity | `order.item_qty` | number | ✅ | ✅ |
| | Individual Item Quantity | `order.max_line_qty` | number | ✅ | ✅ |
| | Payment Method | `order.gateway` | choice (names seen on orders + free text) | — | ✅ |
| **Risk Assessment** | Shopify Risk Level | `risk.shopify` | choice: Low, Medium, High | — | ✅ |
| | Billing vs. Shipping Address Mismatch | `risk.address_mismatch` | bool | — | ✅ |
| | Billing vs. Shipping Country Mismatch | `risk.country_mismatch` | bool | — | ✅ |
| | Shopify Protect Eligibility | `risk.protect` | choice (values from the API) | — | ✅ |
| **Velocity** | Orders from Same IP | `velocity.ip` | window-count | — | ✅ |
| | Orders from Same Email | `velocity.email` | window-count | — | ✅ |
| | Orders from Same Phone | `velocity.phone` | window-count | — | ✅ |
| **Fraud History** | Matches Past Fraud (email, phone or address) | `fraud.match` | bool | — | ✅ |

**Where the harder fields come from (F08)**

| Field | Source |
|---|---|
| **Customer Chargebacks** | Disputes on the customer's previous orders. Needs the disputes read scope. Data exists mainly for Shopify Payments stores; elsewhere the field is empty (and so never matches). |
| **Shopify Protect Eligibility** | The order's Shopify Protect status. Only some stores and regions have it; hide the field when the API returns nothing for the store. |
| **Cross-customer counts** | Number of **different** customers (by customer ID, else email hash) who used the same name / phone / shipping address hash within the window, from `OrderCheck`. |
| **Velocity** | Number of orders with the same IP / email hash / phone hash within the window, excluding the current order. |

## Appendix B — Action catalog

| List | Action | Kind | Options | Notes |
|---|---|---|---|---|
| Checkout | Allow | main | — | Stops the list; nothing hidden |
| Checkout | Hide payment methods | main | method names (default "Cash on Delivery") | Never hides every method |
| Post-order | Allow | main | — | Stops the list; Tag / Email still run if ticked |
| Post-order | Hold fulfillment | main | — | Fulfilment hold + tag `cs-review`; shows under Orders › In review |
| Post-order | Cancel | main | Refund payment, Restock items, Notify customer | Tag `cs-cancelled` |
| Post-order | Tag | information | Order / Customer / Both + tags | |
| Post-order | Email notification | information | — | One email to the alert address |

Only the **first matching enabled rule** in a list acts. Test-mode rules only log.

## Appendix C — Tags, attributes and names

| Name | Kind | Set by | Used by |
|---|---|---|---|
| `_cs_did` | cart attribute | store script (F03) | order intake (F08) |
| `_cs_flags` | cart attribute | store script on MONITOR (F03) | checkout function (F09), order intake (F08) |
| `_cs_did` | cookie | store script (F03) | visitor check (F02) |
| `cs-trusted` | customer tag | allowlist (F06), Approve + trust (F12) | checkout function skips (F09) |
| `cs-fraud` | customer + order tag | Mark as fraud (F12), templates | checkout template "fraud customers" |
| `cs-review` | order tag | review action (F10) | removed on Approve (F12) |
| `cs-cancelled` | order tag | cancel action (F10) | reporting |
| `$app:cart-sentinel` / `config` | metafield on payment customization | rule sync (F09) | checkout function |
| `$app:cart-sentinel` / `public` | shop metafield | settings save (F03) | app embed Liquid |
| `/apps/cs/check` | app proxy path | — | store script |

## Appendix D — Open questions

| # | Question | Needed by |
|---|---|---|
| 1 | Final app name (working name: Cart Sentinel) | F16 / listing |
| 2 | Which Shopify plans support payment customization for an App Store app | F09 |
| 3 | Does the app proxy pass the real visitor IP? | F02 |
| 4 | Does "Pay now" mean a payment method at checkout? (assumed yes) | F09 |
| 5 | Can hashed fraud-list entries be kept after a `customers/redact` request? (legal advice) | F14 |
| 6 | Plan names, prices and limits | F15 |
| 7 | Real DOM markers of PPSpy, Koala Inspector, ShopHunter etc. | F04 |
