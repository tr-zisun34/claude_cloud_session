# Architecture — Behavioral & Bot Shield

Companion to `requirements.md`. This file is the implementation contract:
stack, components, data flow, schemas, APIs, and the scoring algorithm, fixed
enough that Claude Code can generate against it without re-deriving design
decisions.

---

## 1. Component Overview

```
┌─────────────────────┐
│ Storefront           │  Shopify Theme App Extension
│ (collector.js, <5KB)│  → collects signals, posts to Edge Worker
└──────────┬───────────┘
           │ POST /score  (signals + session id)
           ▼
┌──────────────────────┐
│ Edge Worker           │  Cloudflare Workers (or Vercel Edge)
│ - ASN/IP lookup       │
│ - rate counter (KV)   │
│ - scoring function    │
│ - mitigation decision │
└──────────┬────────────┘
           │ async write (fire-and-forget)
           ▼
┌──────────────────────┐        ┌───────────────────────┐
│ Backend API            │◄──────│ Admin Dashboard         │
│ (Node/Express or        │      │ Shopify Remix + Polaris │
│  Remix server routes)   │─────►│ embedded app            │
│ - Postgres              │      └───────────────────────┘
│ - whitelist/settings    │
└──────────┬───────────────┘
           │ webhooks
           ▼
┌──────────────────────┐
│ Shopify Admin API     │  orders/create, customers, app/uninstalled
└──────────────────────┘
```

**Why split Edge Worker from Backend API**: the scoring decision must return
in single-digit milliseconds and must not depend on a round trip to a
regional Postgres instance. The Edge Worker scores and decides synchronously
against fast key-value state (rate counters, whitelist cache); it writes the
resulting log row to the Backend API asynchronously so a DB hiccup never adds
latency to a real shopper's page load. This directly satisfies NFR-1 and
NFR-3 in `requirements.md`.

---

## 2. Tech Stack (decisions, not options)

| Layer | Choice | Reasoning |
|---|---|---|
| Storefront collector | Vanilla JS, no framework | Must stay under 5KB (FR-2/NFR-2); a framework runtime alone would blow the budget |
| Edge scoring | Cloudflare Workers + Workers KV | Sub-10ms cold start, built-in IP/ASN context (`request.cf`), global edge presence |
| Admin app | Remix + Shopify Polaris + App Bridge | Shopify's own recommended stack for embedded apps; first-class session/auth handling |
| Backend DB | Postgres (e.g. Neon/Supabase/RDS) | Relational fit for sessions/whitelist/settings; mature tooling |
| Backend API | Remix server routes (same app as dashboard) | Avoids running a second service for v1; dashboard and API share auth/session code |
| Queue (log writes) | Cloudflare Queues (or plain async `fetch` with retry) | Decouples edge decision from DB write latency |

---

## 3. Data Flow (request lifecycle)

1. Visitor loads a storefront page → `collector.js` loads async, begins sampling signals, assigns/reads a `fingerprint_id` (stored in a first-party cookie, 1-year expiry).
2. On a scoring-relevant event (page view, add-to-cart, form submit), the collector POSTs a signal payload to `/score` on the Edge Worker, including `fingerprint_id`, `shop_domain`, event type, and the raw signal values from FR-3/FR-4.
3. Edge Worker:
   a. Checks Workers KV whitelist cache for `fingerprint_id` or request IP → if present and not expired, short-circuit to `ALLOW`, skip scoring.
   b. Resolves ASN/IP classification from `request.cf.asOrganization` / IP range tables.
   c. Reads/increments the request-rate counter for this `fingerprint_id` in KV (sliding 5s window).
   d. Computes Risk Score per §4.
   e. Looks up merchant's tier thresholds + action overrides (cached from Backend API, refreshed every 60s) to decide the Mitigation Action.
   f. Returns the action to the collector (`{ action: "ALLOW" | "CHALLENGE" | "DECEIVE" | "BLOCK", score, tier }`).
   g. Fires an async write to the Backend API's `/ingest` endpoint with the full score breakdown (does not block the response in step f).
4. Collector enforces the action client-side (render challenge widget, or the storefront theme extension swaps in deceptive data server-side for DECEIVE — see §6) or the Edge Worker itself intercepts and rewrites the response body for BLOCK/DECEIVE cases where it's acting as a reverse-proxy-style intermediary.
5. Dashboard reads aggregated/raw rows from Postgres via Remix server routes.

---

## 4. Scoring Algorithm (fixed v1 weights — merchant-adjustable via Settings)

Each signal contributes points; sum is clamped to [0, 100]. This table is the
default seed data for the `signal_weights` config (see §5), not a hardcoded
constant — but these are the launch values.

| Signal | Condition | Points |
|---|---|---|
| `webdriver_flag` | `navigator.webdriver === true` | +40 |
| `headless_ua` | User-Agent matches known headless pattern list | +25 |
| `canvas_cluster` | Canvas/WebGL hash seen ≥20 times across distinct fingerprints in rolling 24h | +20 |
| `no_mouse_before_click` | Zero mousemove samples recorded before first click/submit event | +15 |
| `untrusted_click` | `event.isTrusted === false` on the action that triggered scoring | +30 |
| `robotic_typing` | Keystroke interval stddev < 15ms across ≥5 characters | +15 |
| `impossible_timing` | Time from first paint to add-to-cart/checkout < 200ms | +25 |
| `datacenter_ip` | ASN classified as hosting/datacenter (not residential/mobile) | +20 |
| `tor_exit` | IP matches current Tor exit node list | +25 |
| `request_burst` | >20 scoring-relevant requests from same fingerprint in 5s | +20 |
| `spy_extension_marker` | DOM marker from signature list (FR-4/FR-25) detected | +50, and tag `session_type = "scraper"` |

**Tier boundaries (defaults, merchant-configurable, FR-26):**
- `score <= 30` → LOW
- `31 <= score <= 70` → MEDIUM
- `score >= 71` → HIGH

**HIGH sub-classification (FR-12):**
- If `session_type == "scraper"` OR (no checkout/login endpoint hit AND page-view rate above threshold) → `DECEIVE`
- Else (checkout or login endpoint hit, or `request_burst` triggered on those endpoints specifically) → `BLOCK`

**MEDIUM → CHALLENGE failure path:** a MEDIUM session that fails the managed
challenge is re-scored with `+30` added (challenge_failed signal), which will
normally push it into HIGH; it is then classified by the same rule above.

---

## 5. Database Schema (Postgres)

```sql
-- One row per installed shop
CREATE TABLE shops (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_domain     TEXT UNIQUE NOT NULL,
  access_token    TEXT NOT NULL,            -- encrypted at rest
  mode            TEXT NOT NULL DEFAULT 'audit', -- 'audit' | 'enforcement'
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  uninstalled_at  TIMESTAMPTZ
);

-- Per-shop tunables; one row per shop
CREATE TABLE settings (
  shop_id                 UUID PRIMARY KEY REFERENCES shops(id) ON DELETE CASCADE,
  low_max_score           INT NOT NULL DEFAULT 30,
  medium_max_score        INT NOT NULL DEFAULT 70,
  high_action_default     TEXT NOT NULL DEFAULT 'auto', -- 'auto' | 'challenge' | 'block'
  signal_weights          JSONB NOT NULL DEFAULT '{}',   -- overrides of §4 defaults
  enabled_signals         JSONB NOT NULL DEFAULT '{}',   -- { "tor_exit": false, ... }
  spy_signatures          JSONB NOT NULL DEFAULT '[]',   -- merchant-editable DOM marker list
  auto_trust_customers    BOOLEAN NOT NULL DEFAULT true, -- FR-23
  notify_email            TEXT,
  notify_threshold_per_hr INT NOT NULL DEFAULT 100,
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Every scored session event (append-only, partition by shop_id + month if volume demands)
CREATE TABLE session_events (
  id              BIGSERIAL PRIMARY KEY,
  shop_id         UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
  fingerprint_id  TEXT NOT NULL,
  ip              INET NOT NULL,
  asn             TEXT,
  page_url        TEXT NOT NULL,
  event_type      TEXT NOT NULL,          -- 'pageview' | 'add_to_cart' | 'checkout_start' | 'form_submit'
  score           INT NOT NULL,
  tier            TEXT NOT NULL,          -- 'low' | 'medium' | 'high'
  triggered_signals JSONB NOT NULL,       -- ["webdriver_flag","datacenter_ip",...]
  action_taken    TEXT NOT NULL,          -- 'allow' | 'challenge' | 'deceive' | 'block'
  audit_mode      BOOLEAN NOT NULL,       -- was this logged-only?
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_session_events_shop_time ON session_events (shop_id, created_at DESC);
CREATE INDEX idx_session_events_fingerprint ON session_events (shop_id, fingerprint_id);

-- Manual overrides / trust list (FR-21, FR-22)
CREATE TABLE whitelist_entries (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_id         UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
  match_type      TEXT NOT NULL,          -- 'fingerprint' | 'ip' | 'customer_id'
  match_value     TEXT NOT NULL,
  reason          TEXT,
  created_by      TEXT NOT NULL DEFAULT 'merchant', -- 'merchant' | 'auto_customer_match'
  expires_at      TIMESTAMPTZ,            -- NULL = permanent
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_whitelist_unique ON whitelist_entries (shop_id, match_type, match_value);

-- Customer-reported false positives (FR-24)
CREATE TABLE block_reports (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_id         UUID NOT NULL REFERENCES shops(id) ON DELETE CASCADE,
  session_event_id BIGINT REFERENCES session_events(id),
  contact_email   TEXT,
  message         TEXT,
  resolved        BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 6. API Contracts

### Edge Worker

**`POST /score`** (called by `collector.js`)
```jsonc
// request
{
  "shop_domain": "example.myshopify.com",
  "fingerprint_id": "fp_abc123",
  "event_type": "add_to_cart",
  "page_url": "/products/widget",
  "signals": {
    "webdriver": false,
    "canvas_hash": "9f2a...",
    "mouse_samples": 14,
    "is_trusted": true,
    "keystroke_intervals_ms": [120, 98, 140],
    "ms_since_first_paint": 4200,
    "dom_markers": []
  }
}
// response
{ "action": "ALLOW", "score": 12, "tier": "low" }
```

**`POST /challenge/verify`** — internal, called after a managed-challenge widget completes; returns updated action per §4's failure path.

### Backend API (Remix server routes, consumed by Edge Worker + Dashboard)

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/ingest` | Edge Worker's async log write (session_events insert) |
| GET | `/api/sessions?tier=&from=&to=&page=` | Dashboard log, paginated/filterable (FR-16) |
| GET | `/api/stats?window=24h` | Dashboard summary tiles (FR-15) |
| GET | `/api/settings` / `PUT /api/settings` | Read/update tunables (FR-18/19/26/27/28) |
| GET | `/api/whitelist` / `POST /api/whitelist` / `DELETE /api/whitelist/:id` | Override management (FR-21/22) |
| POST | `/api/block-report` | Shopper-submitted "this was a mistake" (FR-24) |
| GET | `/api/export?from=&to=` | CSV export (FR-20) |

**Webhooks (Shopify → Backend API):**
- `orders/create` → used to auto-add `customer_id` to whitelist when `auto_trust_customers` is enabled (FR-23)
- `app/uninstalled` → mark shop inactive, stop billing, schedule data deletion per NFR-4 retention policy

---

## 7. Mitigation Execution Detail

- **CHALLENGE**: Edge Worker returns `action: "CHALLENGE"`; `collector.js` renders an invisible managed-challenge widget (e.g. Cloudflare Turnstile) client-side. On pass, a signed short-lived pass token is sent to `/challenge/verify` which re-scores at LOW. On fail, re-score path in §4 applies.
- **DECEIVE**: for scraping-pattern HIGH sessions hitting product/collection pages, the Theme App Extension's server-side rendering hook (or a Worker-level response rewrite on the Storefront API/JSON responses the theme fetches) substitutes `inventory_quantity` and/or `price` fields before the page renders. Real checkout/cart totals are never altered — deception is cosmetic/display-only, never applied to an actual transaction path, to avoid any real financial discrepancy.
- **BLOCK**: Edge Worker returns `action: "BLOCK"`; `collector.js` redirects to a configured 403/landing page that includes the contact link required by FR-24.

---

## 8. Key Design Decisions (ADR-style)

**D1 — Fail open, always.** Any exception, timeout, or missing dependency in
the scoring path defaults to `ALLOW`. Rationale: NFR-3; a bot-blocker that can
take down the store on its own failure is a worse outcome than missing some
bots.

**D2 — Fingerprint, not IP, is the primary key for trust decisions.** IPs
rotate (mobile networks, VPNs); whitelisting an IP alone is fragile. Both are
supported (`whitelist_entries.match_type`), but fingerprint is preferred in
UI copy and default behavior.

**D3 — Deception never touches the real transaction.** `DECEIVE` only
rewrites display-layer data (storefront-rendered price/inventory shown to a
scraper). Cart, checkout, and order creation always use real Shopify data.
This avoids the app ever being the cause of an actual pricing/inventory bug.

**D4 — Checkout pages are out of scope for v1 scoring.** Shopify's checkout
extensibility (Shopify Functions / Checkout UI Extensions) is a separate,
more constrained surface than Theme App Extensions. Scoping v1 to storefront
pages only avoids a dependency on unverified checkout-extensibility
capabilities blocking the whole project. Revisit in v2 (see
`requirements.md` §6).

**D5 — Settings cache, not live DB read, on the hot path.** The Edge
Worker never queries Postgres synchronously; it reads a KV-cached copy of
`settings` and `whitelist_entries`, refreshed every 60s via a scheduled
Worker. This trades up to 60s of staleness on settings changes for
guaranteed low latency on every scored request (supports NFR-1).

---

## 9. Security & Privacy Notes (implementation-level, see NFR-4/NFR-5 for policy)

- `access_token` and any Shopify API credentials encrypted at rest (e.g. via KMS-backed column encryption), never logged.
- Raw mouse-coordinate/keystroke arrays are used in-memory for scoring only and are **not** persisted; only the derived boolean/numeric signals in `triggered_signals` are stored.
- `fingerprint_id` is a derived hash, not reversible to raw device data, and is never joined to `customers` PII except through the explicit, shop-owned `whitelist_entries.match_type = 'customer_id'` path (FR-23), which the merchant controls via a setting.

---

## 10. Build Order (maps to `requirements.md` §7 MVP/Phase split)

1. Shopify app scaffold (Remix + Polaris) + Postgres schema + auth
2. Theme App Extension collector.js (FR-1–6) in isolation, logging to console only
3. Edge Worker scoring function (§4) against mocked signal payloads, unit-tested
4. Wire collector → Edge Worker → `/api/ingest`, confirm end-to-end logging
5. Dashboard: session log + stats tiles (FR-15/16)
6. Mitigation execution: CHALLENGE, then BLOCK, then DECEIVE (in that order of complexity)
7. Whitelist/override UI + auto-customer-trust webhook (FR-21–24)
8. Settings UI for thresholds/signal toggles (FR-18/19/26–28)
9. Bot-simulation + false-positive test pass (see project test plan)
10. Shopify App Store listing + submission
