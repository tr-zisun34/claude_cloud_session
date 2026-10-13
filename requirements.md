# Requirements — Behavioral & Bot Shield

Shopify app that scores storefront visitors by behavior (not just IP/country) and
responds proportionally — allow, silent challenge, deceive, or block — to stop
checkout bots, inventory/price scrapers, competitor spy extensions, and
credential stuffing, without blocking real shoppers.

This document is the source of truth for scope. `architecture.md` covers how
it's built; this file covers what it must do.

---

## 1. Glossary

| Term | Meaning |
|---|---|
| **Session** | One visitor's browsing activity on the storefront, identified by a fingerprint + cookie, from landing to leaving or converting |
| **Signal** | A single measurable fact about a session (e.g. `navigator.webdriver === true`) |
| **Risk Score** | 0–100 integer, sum of weighted signals for a session |
| **Risk Tier** | LOW / MEDIUM / HIGH, derived from Risk Score against merchant-configurable thresholds |
| **Mitigation Action** | What happens to a session at a given tier: `ALLOW`, `CHALLENGE`, `DECEIVE`, `BLOCK` |
| **Audit Mode** | App computes scores and logs actions it *would* take, but never actually challenges/deceives/blocks |
| **Enforcement Mode** | App actually applies the mitigation action |
| **Override** | Merchant manually reclassifying a flagged session/IP/fingerprint as trusted |
| **Fingerprint** | A derived, non-PII identifier for a device/browser, stable across a session and (best-effort) across sessions |

---

## 2. Personas

- **Merchant Admin** — installs the app, configures it, reviews the dashboard, performs overrides. Primary user of the Admin Dashboard.
- **Shopper** — the storefront visitor being scored. Never sees the system unless challenged or blocked.
- **Bot/Scraper Operator** — the adversary. Not a user, but their behavior defines acceptance criteria for detection.

---

## 3. Functional Requirements

### 3.1 Telemetry Collection (Storefront)

- **FR-1**: The app MUST inject a client-side script via Shopify Theme App Extension on all storefront pages (home, collection, product, cart; checkout is out of scope for v1 — see §6).
- **FR-2**: The script MUST be ≤5KB gzipped and load asynchronously, never blocking page render.
- **FR-3**: The script MUST collect, per session, without prompting the user:
  - `navigator.webdriver` value
  - Presence/absence of `navigator.plugins`, `navigator.languages`
  - A canvas + WebGL render hash
  - Timestamped mouse-move sample points (throttled, not every pixel)
  - `isTrusted` value on click/submit events
  - Keystroke interval timestamps on any `<input>` the visitor types into
  - Timestamp of first paint and timestamp of any "Add to Cart" / "Checkout" action
- **FR-4**: The script MUST scan the DOM for known marker patterns left by inventory/price scraping extensions (configurable signature list, see §3.5) and report matches.
- **FR-5**: The script MUST attach a short-lived signed token (`x-bot-score` or session cookie) carrying a session ID, not the raw signal data, to outgoing requests.
- **FR-6**: If the script fails to load or throws, the page MUST render normally with no visible error (fail silent, fail open).

### 3.2 Edge Scoring

- **FR-7**: On each scored request, the edge service MUST compute:
  - Client-reported signal score (from FR-3/FR-4 data)
  - Network signal score: ASN/IP classification (residential / datacenter / hosting / known Tor exit) and request-rate-per-session
- **FR-8**: The edge service MUST combine both into one Risk Score (0–100) per the weighting table in `architecture.md` §4.
- **FR-9**: Scoring MUST complete and return an action within 50ms server-side p95; if it cannot, the request MUST be treated as `ALLOW` (fail open — see NFR-3).
- **FR-10**: Risk Score, triggered signals, and resulting action MUST be persisted per session for the dashboard and override log.

### 3.3 Mitigation

- **FR-11**: Each Risk Tier MUST map to exactly one default Mitigation Action, merchant-overridable per tier:
  - LOW → `ALLOW`
  - MEDIUM → `CHALLENGE` (invisible managed challenge; on failure, re-score as HIGH)
  - HIGH → `DECEIVE` or `BLOCK`, chosen by sub-classification (see FR-12)
- **FR-12**: Within HIGH tier, the app MUST distinguish:
  - **Scraping pattern** (high page-view rate, no checkout-page hits, datacenter IP) → `DECEIVE`: serve fabricated inventory (always "1 in stock") and/or altered prices on the response the scraper reads.
  - **Checkout/credential pattern** (checkout or login endpoints hit, impossible timing, high request rate) → `BLOCK`: 403 or redirect to a configured URL.
- **FR-13**: `DECEIVE` responses MUST NOT be served on a session's first page view — only after enough signal has accumulated to cross the HIGH threshold — so a legitimate first-time visitor is never shown fake data.
- **FR-14**: The merchant MUST be able to toggle **Audit Mode** globally; in Audit Mode, actions are computed and logged but the storefront response is always the unmodified real page.

### 3.4 Admin Dashboard

- **FR-15**: Dashboard MUST show, in near-real-time: total sessions scored (24h/7d/30d), count per risk tier, count per mitigation action taken, and a live feed of the most recent HIGH-tier sessions.
- **FR-16**: Dashboard MUST show a sortable/filterable log of flagged sessions with: timestamp, IP, fingerprint ID, score, triggered signals, action taken, page URL.
- **FR-17**: Dashboard MUST let the merchant switch Audit Mode ⇄ Enforcement Mode with a single control, with a confirmation step before first enabling Enforcement Mode.
- **FR-18**: Dashboard MUST expose per-tier action overrides (e.g. force HIGH tier to always `CHALLENGE` instead of `BLOCK`).
- **FR-19**: Dashboard MUST expose per-signal enable/disable toggles and (advanced) weight adjustment.
- **FR-20**: Dashboard MUST support CSV export of the flagged-session log for a given date range.

### 3.5 Override / Fallback (fail-safe — non-negotiable for v1)

- **FR-21**: From the flagged-session log, the merchant MUST be able to click "Trust this visitor," which:
  - Adds the session's fingerprint (and, optionally, IP) to a Whitelist
  - Immediately reduces that fingerprint's/IP's future score to LOW regardless of signals, until the whitelist entry expires or is removed
- **FR-22**: Whitelist entries MUST support an optional expiry (e.g. 24h, 7d, permanent) and a free-text reason field.
- **FR-23**: The app MUST provide a setting to auto-whitelist any visitor matched to a Shopify customer account with ≥1 prior completed order (via order/customer webhook data), bypassing scoring entirely when enabled.
- **FR-24**: Any `BLOCK` response page MUST include a visible contact link/email the shopper can use to request a manual review; submissions MUST land in the merchant's flagged-session log tagged "customer-reported."
- **FR-25**: Known spy-extension signature list (FR-4) MUST be editable/extendable by the merchant without a code deploy (stored as config, not hardcoded).

### 3.6 Settings & Configuration

- **FR-26**: Merchant MUST be able to set the LOW/MEDIUM/HIGH score boundaries (defaults 0–30 / 31–70 / 71–100).
- **FR-27**: Merchant MUST be able to allowlist specific ASNs/IP ranges (e.g. their own office, known good crawlers) globally, bypassing scoring.
- **FR-28**: Merchant MUST be able to set notification preferences (email) for spikes in HIGH-tier traffic over a rolling window.

---

## 4. Non-Functional Requirements

- **NFR-1 (Latency)**: No legitimate (LOW tier) visitor experiences more than 10ms added latency attributable to scoring, measured server-side.
- **NFR-2 (Footprint)**: Storefront JS payload ≤5KB gzipped; no synchronous blocking script tags.
- **NFR-3 (Fail-open)**: Any failure in the telemetry collector, edge scoring service, or its dependencies MUST default to `ALLOW`. The app must never be able to take down or block an entire storefront due to its own outage.
- **NFR-4 (Privacy)**: No raw signal data (keystrokes, exact mouse coordinates) is retained beyond the scoring window needed to compute and log the score breakdown; only derived scores/flags are stored long-term. Fingerprints must not be derived from or stored alongside PII (name, email, address) except where explicitly linked via FR-23's opt-in customer match.
- **NFR-5 (Transparency)**: The merchant's privacy policy obligations (what data is collected, retention period, legal basis) must be documented and merchant-facing before Enforcement Mode can be enabled for the first time (a one-time acknowledgment step).
- **NFR-6 (Availability)**: Edge scoring service target 99.9% uptime; dashboard target 99.5% (dashboard downtime does not affect storefront behavior).
- **NFR-7 (Scalability)**: Must handle traffic spikes (flash sales) of at least 50x baseline RPS without increasing per-request latency beyond NFR-1.

---

## 5. User Stories (sample set — expand per sprint)

**Epic: Detection**
- As a merchant, I want headless browsers automatically scored HIGH, so that basic scraping bots never need a human to review them.
  - AC: A Playwright session with default settings reaches HIGH tier within its first request, with `navigator.webdriver` logged as a triggered signal.
- As a merchant, I want a shopper using a corporate VPN to not be blocked, so that I don't lose a legitimate sale.
  - AC: A session with only "datacenter-adjacent ASN" triggered and normal mouse/click behavior stays at MEDIUM, receives a `CHALLENGE`, and passes it silently.

**Epic: Mitigation**
- As a merchant, I want detected scrapers to see fake stock numbers instead of a block page, so competitors can't tell they've been caught.
  - AC: A session classified HIGH + scraping-pattern receives `inventory_quantity: 1` on all variants regardless of real stock, verified via test harness.

**Epic: Override**
- As a merchant, I want to unblock a real customer who got flagged, so I don't lose their trust or their order.
  - AC: Clicking "Trust this visitor" on a logged session results in that fingerprint scoring LOW on its next request within 5 seconds.

**Epic: Reporting**
- As a merchant, I want to see how many threats were stopped this week, so I can justify the app's cost.
  - AC: Dashboard summary tile shows count of BLOCK + DECIEVE actions for the last 7 days, refreshed at least every 5 minutes.

*(Full backlog to be expanded into one story + AC per FR above during sprint planning.)*

---

## 6. Explicitly Out of Scope (v1)

- Scoring or modifying behavior on Shopify's native Checkout pages (Shopify Functions/Checkout UI Extensions are a separate, more restricted extensibility surface — treated as a v2+ investigation, not a v1 commitment).
- Machine-learning-based score tuning (v1 uses fixed, merchant-adjustable weights only).
- Multi-store / agency-level cross-store analytics.
- Native mobile app.

## 7. Open Questions

- Exact legal/compliance review of fingerprinting under GDPR/CCPA for EU/CA merchants — needs sign-off before public launch, not assumed resolved by this doc.
- Confirm current Shopify extensibility limits for checkout-adjacent scoring (API version drift risk — verify against latest Shopify docs at implementation time, not against this document).
- Pricing/plan tiers — not yet decided, not a blocker for MVP build.
