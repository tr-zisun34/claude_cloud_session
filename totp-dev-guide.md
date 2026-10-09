# Checkout Guard - Authenticator app (TOTP) development guide

How to add **authenticator app codes (TOTP)** as a second way to verify, next to the emailed code. It covers the decisions, data model, security rules, every new file and function, the PR plan with "Done when" checks, and the tests.

Read with: [CLAUDE.md](../CLAUDE.md) (architecture, security rules, coding standards) and the spec (Features 5-8). Where this guide and CLAUDE.md disagree on general rules, CLAUDE.md wins.

> **Plan revision.** An earlier draft put linking the authenticator app on a **customer account UI extension** (a full page in customer accounts). This version drops the extension: **everything — linking, verifying and removing — happens inside the existing checkout-guard pop-up**, over the app proxy we already use. This removes a whole surface (no extension bundle, no `network_access` approval, no session-token/`dest` handling) and every Shopify-platform unknown that came with it. The trade-off is that customers manage the app **at the checkout moment only**; email stays the always-available recovery path, so nothing is lost.

---

## 1. What we're building

Today a customer who must verify gets a **6-digit code by email**. With this feature, a customer can **link an authenticator app** (Google Authenticator, Microsoft Authenticator, 1Password, Authy…) once, **right from the verification pop-up**. From then on, the pop-up asks for the **code shown in the app** and shows a green "Authenticator app connected" badge. That's faster and works without waiting for email.

Everything after a correct code stays the same: **the same `issuePass()`, the same pass lifetime, the same checkout function.** The checkout function doesn't change at all.

```
Pop-up "Verification required"  (the existing checkout-guard modal)
  ├─ customer has a linked app ─→ ✓ "Authenticator app connected" (green badge)
  │                               "Enter the code from your authenticator app"   [Email me a code instead]
  └─ no linked app ─────────────→ Send code (email), unchanged
                                   + [Connect to authenticator app]  ← button at the top
                                        │ press
                                        ▼
                                   QR + manual key → enter first code → confirm → linked
            │ correct code (either way)
            ▼
       issuePass() → pass metafield on the customer → checkout allowed
```

The connect button and the green badge are **mutually exclusive**: both come from one server-reported flag (does this customer have an `active` app?). Once linked, the pop-up never asks to connect again — it shows the badge and goes straight to the code field.

### Goals
- Customers can link, use and remove an authenticator app themselves, from the pop-up.
- The pop-up uses the app when the customer has linked one, with **email always available as a fallback**.
- Merchants can turn the option on or off.
- The security level is the same as or better than email codes.

### Not in scope
- **Making TOTP mandatory:** email stays available.
- **SMS codes, passkeys and recovery codes:** email is the recovery path.
- **A standalone account page to manage the app** (could be added later; see §7.4). Management happens in the pop-up.
- Merchant-side reset of a customer's app (could be added later in the admin).
- **Guests:** they still log in first, as today.

---

## 2. Decisions

These are the recommended defaults; **confirm them with the team lead before PR 1**.

| # | Decision | Default | Why |
|---|---|---|---|
| T1 | Where customers link the app | **Inside the checkout-guard pop-up** (storefront), via a "Connect to authenticator app" button | No new surface; reuses the app proxy and the identity we already trust there |
| T2 | Optional or required | **Optional for customers**; the merchant switch only allows or hides the option | No customer gets locked out |
| T3 | Email fallback | **Always available** in the pop-up ("Email me a code instead") | Lost or changed phone; no recovery codes to manage |
| T4 | Removing the app | Requires a **current authenticator code or an emailed code** | A stolen account session alone can't remove it; a lost phone can still remove it by email |
| T5 | Library | **None**: TOTP (RFC 6238) is about 40 lines on Node's `crypto`; the QR is rendered **server-side as an SVG** (see 6.4) | No new dependencies, and the storefront JS stays tiny |
| T6 | Code format | **6 digits, 30-second steps, SHA-1**, accepting the previous and next step (±1) | What every authenticator app supports by default |
| T7 | Where management lives | **The pop-up, at checkout time only** | Email recovery (T3) means a lost phone is handled at the next checkout; no account page needed |
| T8 | Who identifies the customer | The **app proxy** request (`logged_in_customer_id`, via `authenticateProxy`) | Same trusted identity the email-code flow already uses; no session token or `dest` to parse |

---

## 3. Where code runs

| Place | New or changed | Talks to |
|---|---|---|
| **Storefront pop-up** (`checkout-guard-code.js`) | **Changed:** connect screen (QR + confirm), TOTP screen, green badge, "Email me a code instead" | App server via the **app proxy**, as today |
| **App server** | New `lib/totp/`, new proxy routes (`proxy.totp.*`), changed `proxy.requirements`, changed `issuePass` | Postgres; Admin API (shop name) |
| **Checkout function** | **No change** | - |

There is **no customer account extension**. Every call the pop-up makes goes through the app proxy (`/apps/step-up/…`), which Shopify signs and which carries `logged_in_customer_id`. `authenticateProxy(request)` turns that into `identity.customerId` — the same thing the email-code routes already rely on. A logged-out shopper has no `customerId` and gets `login_needed`, exactly as today.

---

## 4. Data model

Unchanged from the extension plan — the tables don't care which surface drove them.

### New table `CustomerAuthenticatorApp`

```prisma
enum AuthenticatorStatus {
  pending // QR shown, not yet confirmed with a first code
  active  // used at checkout
}

model CustomerAuthenticatorApp {
  id              String              @id @default(cuid())
  shop            String
  customerId      String              // gid://shopify/Customer/123
  status          AuthenticatorStatus @default(pending)
  encryptedSecret String              // "v1:<iv>:<tag>:<ciphertext>", base64 parts; see 5.2
  lastUsedStep    Int?                // highest 30 s step accepted; blocks reusing a code
  createdAt       DateTime            @default(now()) @db.Timestamptz(3)
  confirmedAt     DateTime?           @db.Timestamptz(3)

  @@unique([shop, customerId])        // one app per customer per store
}
```

- **One row per customer per store.** "Connect" replaces a `pending` row; an `active` row must be removed (T4) before linking again.
- `pending` rows older than **15 minutes** are ignored when confirming and deleted by the nightly cleanup.
- Every query filters by `shop` (`byShop`), like every other table.

### Change to `StoreCheckoutRules`

```prisma
authenticatorAppAllowed Boolean @default(false) // merchant switch (section 9)
```

It isn't part of the compiled rules: the checkout function doesn't care **how** a customer verified.

Migration: `npx prisma migrate dev --name authenticator_app`.

---

## 5. Security rules

These come on top of CLAUDE.md's security rules.

### 5.1 The secret
- **20 random bytes** (`crypto.randomBytes(20)`, 160 bits, as RFC 4226 recommends).
- It's shown **once**, during linking: as a QR code and as base32 text for manual entry. It's never sent again, never logged, and never returned by any "status" call.

### 5.2 Encrypted at rest
Unlike email codes, the secret **can't be hashed**: we need it to compute the expected code.
- AES-256-GCM, key from `.env` `TOTP_ENCRYPTION_KEY` (32 random bytes, base64). A random 12-byte IV per secret.
- Stored as `v1:<iv>:<authTag>:<ciphertext>`. The `v1` prefix allows changing the key later: add `v2` with a new key, decrypt both, and re-encrypt on next use.
- If the key is missing, **TOTP is off**: `requirements` never offers it, and the connect route answers `unavailable`. The app must never crash or store plain secrets.

### 5.3 Checking a code
- Compare with `crypto.timingSafeEqual` (codes as **strings**, so leading zeros are kept).
- Accept steps `current - 1 … current + 1` (about ±30 s clock drift).
- **Replay:** a step at or below `lastUsedStep` is rejected. Save the new step with a **conditional update** (`where lastUsedStep < step or null`), so two parallel requests with the same code can't both win. This is the same pattern as `consumeChallenge`.
- **Try limit:** reuse `checkAndCount` (`app/lib/otp/limits.server.ts`) with key `totp:<customerId>`, **5 tries per hour**, counted **before** comparing (like `reserveAttempt`). A correct code doesn't reset the count.
- Use the **database's** time or the server clock consistently. A 30 s step makes clock drift between app servers harmless.

### 5.4 Who can change what
- Connect, confirm and remove only ever act on the customer in the **verified app-proxy identity** (`identity.customerId` from `authenticateProxy`), never on an ID from the request body.
- **Remove** requires a current authenticator code **or** an emailed code (T4).
- Linking again while one is `active` is refused, so a hijacked session can't silently swap the app.

### 5.5 Logging and privacy
- Log topic, shop and the outcome (`totp ok` / `totp invalid` / `totp locked`) only. Never secrets, codes, otpauth URIs or emails.
- Privacy webhooks delete or report the row (section 10).

---

## 6. Server code

### 6.1 `app/lib/totp/constants.ts`
```ts
export const TOTP_DIGITS = 6;
export const TOTP_STEP_SECONDS = 30;
export const TOTP_ALLOWED_DRIFT_STEPS = 1;
export const TOTP_TRIES_PER_HOUR = 5;
export const TOTP_PENDING_MINUTES = 15;
export const TOTP_SECRET_BYTES = 20;
```

### 6.2 `app/lib/totp/totp.ts`: pure, no Shopify or database code (fully unit-testable)
| Function | Does |
|---|---|
| `generateSecret(): Buffer` | `crypto.randomBytes(TOTP_SECRET_BYTES)` |
| `toBase32(secret: Buffer): string` | RFC 4648 base32, no padding; the text the customer can type in |
| `currentStep(now = Date.now()): number` | `Math.floor(now / 1000 / TOTP_STEP_SECONDS)` |
| `codeAt(secret: Buffer, step: number): string` | HOTP (RFC 4226): HMAC-SHA1 over the 8-byte big-endian step → dynamic truncation → `% 10^6` → zero-padded to 6 digits |
| `matchingStep(secret, code, now): number \| null` | Checks steps `current-1 … current+1` with `timingSafeEqual`; returns the matching step or null |
| `otpauthUri({ secret, issuer, label }): string` | `otpauth://totp/<issuer>:<label>?secret=<base32>&issuer=<issuer>&algorithm=SHA1&digits=6&period=30` (URL-encoded) |

**Tests (`totp.test.ts`)**:
- the **RFC 6238 test vectors** for SHA-1 (e.g. secret `12345678901234567890`, time 59 → `287082`);
- a code from the previous and next step matches, two steps away doesn't;
- leading zeros are kept;
- base32 round-trips.

### 6.3 `app/lib/totp/crypto.server.ts`
| Function | Does |
|---|---|
| `isTotpConfigured(): boolean` | `TOTP_ENCRYPTION_KEY` is set and decodes to 32 bytes |
| `encryptSecret(secret: Buffer): string` | AES-256-GCM → `v1:iv:tag:ct` |
| `decryptSecret(stored: string): Buffer` | The reverse; throws on a wrong key, tampering or an unknown version |

**Tests:** round trip; a changed ciphertext or tag throws; a missing key → `isTotpConfigured()` is false.

### 6.4 `app/lib/totp/qr.server.ts`: render the QR ourselves (no `s-qr-code`)
The customer account extension gave us a built-in QR component; the storefront doesn't have one, so the **server** turns the `otpauth://` URI into an **SVG** and the pop-up just drops it in.

| Function | Does |
|---|---|
| `qrSvg(text: string): string` | QR (error-correction M) → a self-contained `<svg>` string, no external request, no `<script>` |

- Keep it dependency-light: a tiny vendored QR encoder, or a small well-reviewed package if CLAUDE.md allows a new dep. **No remote QR image services** (never send the secret off our servers).
- The SVG is sized with `width`/`height`/`viewBox` only; the pop-up injects it with `innerHTML` into a container it controls (the SVG is ours, not customer text — see §8 security note).

**Tests:** a known URI produces a non-empty `<svg>` that a QR decoder reads back to the same URI.

### 6.5 `app/lib/totp/authenticator.server.ts`: the database rules
| Function | Does |
|---|---|
| `getAuthenticatorStatus(shop, customerId)` | `'none' \| 'pending' \| 'active'`, plus `confirmedAt`. Never the secret. |
| `startSetup(shop, customerId, issuer)` | Refuses if `active`. Otherwise creates or replaces the `pending` row with a new encrypted secret. Returns `{ otpauthUri, manualKey }` **once**. |
| `confirmSetup(shop, customerId, code)` | Counts a try (`checkAndCount`) → loads the `pending` row younger than 15 min → `matchingStep` → conditional update to `active`, `confirmedAt = now`, `lastUsedStep = step`. Returns `ok \| invalid \| expired \| locked`. |
| `verifyAuthenticatorCode(shop, customerId, code)` | Counts a try → loads the `active` row → `matchingStep` → **conditional** `lastUsedStep` update (replay check). Returns `ok \| invalid \| locked \| none`. |
| `removeAuthenticator(shop, customerId)` | Deletes the row. The route checks the confirming code first. |

**Tests (`authenticator.server.test.ts`, database mocked like `challenge.server.test.ts`)**:
- setup refused when active;
- setup replaces a pending row;
- confirm with a wrong code, an old pending row, or too many tries;
- a correct code activates;
- verify rejects a replayed step;
- the try is counted before comparing.

### 6.6 App-proxy routes (new) and `requirements` (changed)
Every new route is a thin app-proxy handler, exactly like `proxy.otp.send.tsx`:
```ts
const identity = await authenticateProxy(request);
if (!identity.customerId) return proxyJson({ ok: false, reason: 'login_needed' }, 401);
// act only on identity.customerId and identity.shop — never an id from the body
```
Bodies are parsed with zod. The **issuer** for the QR is the **shop's name** (`shop { name }` via the Admin client the proxy already has), so the customer sees "My Store" in their app, not our app's name.

| Route file | Method | Body → answer |
|---|---|---|
| `proxy.requirements.tsx` *(changed)* | - | When the decision is `verify`: add `hasAuthenticator: boolean` (the customer has an `active` app **and** the merchant switch is on **and** TOTP is configured). The pop-up renders badge-vs-connect from this one flag, with no extra round-trip. |
| `proxy.totp.setup.tsx` | POST | `startSetup` → `{ ok, otpauthUri, qrSvg, manualKey }`, or `{ ok: false, reason: 'not_allowed' \| 'already_active' \| 'unavailable' }` |
| `proxy.totp.confirm.tsx` | POST | `{ code }` → `{ ok }` or `{ ok: false, reason: 'invalid' \| 'expired' \| 'locked' }` |
| `proxy.totp.verify.tsx` | POST | zod `{ code: /^\d{6}$/ }` → `verifyAuthenticatorCode` → `ok` → **`issuePass(shop, customerId, null, timezone, admin)`**. Reasons: `invalid` (+ `triesLeft`), `locked`, `none` (no app linked; the pop-up switches to email), `pass_failed`. |
| `proxy.totp.remove-code.tsx` | POST | sends an **email code** for removing (reuses the OTP send logic: `getCustomerEmail`, limits, `buildCodeEmail`) → `{ ok, challengeId, maskedEmail }` |
| `proxy.totp.remove.tsx` | POST | `{ method: 'authenticator', code }` or `{ method: 'email', challengeId, code }` → checks it (`verifyAuthenticatorCode` or the email `reserveAttempt` → `consumeChallenge` path) → `removeAuthenticator` → `{ ok }` |

- **`issuePass`** gets `codeId: string | null` (a TOTP pass has no email code), and `CustomerVerification.emailVerificationCodeId` becomes optional. Add a `method` column (`email | authenticator`) on `CustomerVerification` for support and data requests.
- **Refactor first (shared logic):** move the email-code send and verify steps out of `proxy.otp.send.tsx` / `proxy.otp.verify.tsx` into `app/lib/otp/send.server.ts` and `verify.server.ts` (`sendEmailCode(shop, customerId, ip)`, `checkEmailCode(shop, customerId, challengeId, code)`). The proxy OTP routes and the new `totp.remove-code` / `totp.remove` then share them (DRY), and the routes stay thin.

---

## 7. Storefront pop-up (the whole customer-facing surface)

All of this lives in **`checkout-guard-code.js`** (`<checkout-guard-code>`), not the 10 KB-limited `checkout-guard.js` (see §8 for the byte budget). Liquid holds the texts as `data-` attributes; locales hold the strings.

### 7.1 States, driven by `hasAuthenticator`
On open, the pop-up already has the `requirements` response. It renders one of:

| State | Shows |
|---|---|
| `hasAuthenticator: true` | A green **"✓ Authenticator app connected"** badge, heading "Enter the code from your authenticator app", the **same 6 boxes with auto-verify on the 6th digit** (posting to `proxy.totp.verify`), and a link button **"Email me a code instead"**. **No connect button.** |
| `hasAuthenticator: false` | Today's email flow (Send code), plus a **"Connect to authenticator app"** button at the top. |
| connecting (after the button → `setup`) | Step 1: the server's **QR SVG** and the **manual key** in groups of 4, with a copy control. Step 2: a 6-digit field (`inputMode="numeric"`, `autocomplete="one-time-code"`), then **Confirm** → `confirm`. On `ok` → go to the TOTP screen (now "connected") and let the customer verify. Errors in an alert banner. |
| removing (optional, from the connected state) | A small menu: enter a current app code, **or** "Email me a code" (`remove-code`) then enter it → `remove`. |

- The badge and the connect button are never shown together — one boolean decides.
- **Trust the server, not a client cache:** render from the latest `requirements`/route response every time the modal opens. If `proxy.totp.verify` ever returns `none` (the app was removed since the modal opened, or the key/row is gone), drop to the email screen and hide the badge. `locked` → show the error and keep the email option.

### 7.2 Flow
1. `requirements` answers `{ decision: 'verify', hasAuthenticator }`.
2. `hasAuthenticator` true → authenticator screen (badge + code boxes); false → email screen + connect button.
3. Correct code (either path) → the same success path as email (repeat the original click / go to checkout).
4. `none` mid-flow → switch to the email screen. `locked` → the error, email still offered.

### 7.3 Files
| File | Change |
|---|---|
| `checkout_guard.liquid` | Texts as `data-` attributes: `connect_cta`, `connected_badge`, `authenticator_heading`, `email_instead`, `authenticator_invalid`, `authenticator_locked`, `setup_scan`, `setup_manual`, `setup_confirm` |
| `locales/en.default.json` | The texts above |
| `checkout-guard-code.js` | The connect screen (drop in the server QR SVG + manual key + confirm), the authenticator screen with the badge, the submit (`fetch` to `/totp/verify` through the proxy path), and the switches to/from email |
| `checkout-guard.css` | The green "connected" badge, the QR container, a link-style button for "Email me a code instead" |

Security as today: **customer-controlled text is set with `textContent` only**, never as HTML. The one exception is the **QR SVG**, which is **our** server output (not customer input) and is injected into a container element we own.

### 7.4 (Optional, later) a customer account page
If we later want "manage my authenticator any time," the same `authenticator.server.ts` functions back a customer-account UI extension page with no server changes. It's intentionally **out of scope** here; email recovery (T3) covers the lost-phone case at the next checkout.

---

## 8. Storefront byte budget

`checkout-guard.js` is at Theme Check's **`AssetSizeJavaScript`** limit (default `threshold_in_bytes: 10000`, i.e. ~10 KB), so **all new UI goes in `checkout-guard-code.js`**, which isn't bound by that rule. The only change to `checkout-guard.js` is reading `hasAuthenticator` off the `requirements` response and handing it to `<checkout-guard-code>` (one small line).

- The QR is a **server-rendered SVG**, so no QR library ships in the storefront bundle.
- **Measure before shipping** (PR 4 "Done when"): `wc -c assets/checkout-guard.js`, or `npx @shopify/theme-check-cli --category performance`. If the one line pushes `checkout-guard.js` over, move that read into `checkout-guard-code.js` instead.

---

## 9. Merchant settings

- **Settings page:** a new card, "Authenticator app (optional for customers)", with a switch: *"Let customers use an authenticator app instead of an email code."* Saved to `authenticatorAppAllowed` through the existing `saveRules` (add it to `rulesFormSchema`, `RulesUpdate`, `useRulesForm`, `DEFAULT_RULES`).
- Turning it **off**: `requirements` sets `hasAuthenticator: false` and the pop-up hides the connect button (email only). Linked apps are **kept**, so turning it back on restores them.
- **Help page:** a section explaining the feature — that customers connect it from the verification pop-up, that a green badge shows when it's connected, and that email always works.
- **No theme/customer-accounts setup for merchants:** there's no extension to place. The pop-up already renders on the storefront.

---

## 10. Privacy, uninstall and cleanup

| Event | Change |
|---|---|
| `customers/redact` | Also delete the customer's `CustomerAuthenticatorApp` row and the `totp:<customerId>` limit counter |
| `customers/data_request` | Also report "authenticator app linked on <confirmedAt>" (never the secret) |
| `shop/redact` | Also delete all rows for the store |
| `app/uninstalled` | Keep the rows (a reinstall keeps customers' apps working); `shop/redact` deletes them after 48 h |
| Nightly cleanup | Delete `pending` rows older than 15 minutes |

All must stay **safe to run twice** (CLAUDE.md webhook rule); add the cases to `app/tests/webhooks.privacy.test.ts`.

---

## 11. Environment

| Name | Value |
|---|---|
| `TOTP_ENCRYPTION_KEY` | 32 random bytes, base64. Generate with `node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"`. **Never commit; never change it without the `v2` migration (5.2)**, or every linked app stops working. |

Add it to CLAUDE.md's Environment table.

---

## 12. PR plan

Branch per PR from `develop`, small and one job each, as before. (No extension PR anymore — everything is server + storefront.)

### PR 1: TOTP core (`feature/totp-core`, ~3 h)
- **Files:**
  - `lib/totp/constants.ts`, `totp.ts`, `crypto.server.ts`, `qr.server.ts`, `authenticator.server.ts` (+ tests);
  - schema: `CustomerAuthenticatorApp`, `AuthenticatorStatus`, `authenticatorAppAllowed`, `CustomerVerification.method`, optional `emailVerificationCodeId`;
  - migration;
  - `issuePass` accepts `codeId: null`;
  - `.env` key.
- **Done when:** the RFC test vectors pass; replay, drift, lock and encryption tests pass; `qrSvg` round-trips; `npm test` and typecheck are green.

### PR 2: Email-code refactor (`feature/otp-shared`, ~1 h)
- **Files:** `lib/otp/send.server.ts`, `verify.server.ts`; `proxy.otp.send.tsx` / `proxy.otp.verify.tsx` call them.
- **Done when:** the existing OTP tests pass unchanged, and the pop-up email flow works as before.

### PR 3: TOTP proxy routes + `requirements` (`feature/totp-proxy`, ~2.5 h)
- **Files:** `proxy.totp.{setup,confirm,verify,remove-code,remove}.tsx` (+ route tests in `app/tests/`); `proxy.requirements.tsx` adds `hasAuthenticator`.
- **Done when:**
  - connect → confirm → `active` works against a dev store with Google Authenticator;
  - verify issues a pass; replayed code rejected; 6th wrong try locks;
  - remove with an app code works, remove with an email code works;
  - linking again while active is refused;
  - every route acts only on `identity.customerId`.

### PR 4: Pop-up UI (`feature/authenticator-popup`, ~4 h)
- **Files:** `checkout-guard-code.js` (connect screen with the server QR SVG, authenticator screen + green badge, email switch), `checkout_guard.liquid`, locales, CSS; one line in `checkout-guard.js` to pass `hasAuthenticator` (byte budget checked).
- **Done when:**
  - a customer with no app sees the connect button; connects; badge turns green; verifies with the app → checkout opens;
  - a returning linked customer sees the badge and the code field, **no connect button**;
  - "Email me a code instead" → the email flow;
  - a customer without an app sees today's flow unchanged;
  - Theme Check passes, and `checkout-guard.js` is under 10 KB.

### PR 5: Merchant switch, privacy, Help (`feature/authenticator-settings`, ~1.5 h)
- **Files:** settings schema, form and card; `proxy.requirements` honours the switch; privacy webhooks + tests; nightly pending cleanup; Help page; CLAUDE.md.
- **Done when:**
  - the switch off → email only, no connect button;
  - the switch on again → linked apps work;
  - `shopify app webhook trigger` for the three privacy topics leaves no authenticator rows behind.

### PR 6: Dev-store test pass (~1.5 h)
Section 13, start to finish; fix what turns up in small PRs.

---

## 13. Test plan (dev store)

| # | Scenario | Expected |
|---|---|---|
| 1 | In the pop-up, press "Connect", scan with Google Authenticator, confirm with the shown code | Linked; green badge; the row is `active` |
| 2 | Confirm with a wrong code 5 times | Locked; the try counter is in `CodeSendLimit` |
| 3 | Start connecting, wait 16 min, confirm | Expired; start again |
| 4 | Returning linked customer hits the limit | Pop-up shows the badge and code field, **no connect button**; correct code → checkout |
| 5 | The same code again within 30 s (second tab) | Rejected (replay) |
| 6 | Phone clock 25 s off | Still accepted (±1 step) |
| 7 | "Email me a code instead" | The email flow, unchanged |
| 8 | Remove with an app code; remove with an email code | Removed; next checkout shows the connect button again |
| 9 | Merchant switch off | Email only; no connect button; linked apps kept |
| 10 | No plan (cancelled) | No pop-up at all (plan gating) |
| 11 | `customers/redact` for that customer | Row and counter gone |
| 12 | App removed server-side between opening the modal and submitting | `proxy.totp.verify` → `none`; pop-up drops to email |

---

## 14. Edge cases

- **The customer's phone time is wrong** by more than 30 s → codes fail. The pop-up should say "make sure your phone's time is set automatically". Email still works.
- **Two staff devices or two apps** scanning the same QR both work (same secret). That's expected, and not a problem.
- **The customer changes their account email** → no effect on the app.
- **Encryption key lost or changed** → decryption fails. Treat it as `none` (fall back to email, hide the badge), log once per shop, never crash the pop-up.
- **The merchant uninstalls and reinstalls within 48 h** → rows are kept, and apps keep working.
- **A customer is deleted in Shopify** → `customers/redact` arrives later and removes the row.
- **A customer wants to manage the app but isn't checking out** → there's no page (T7). They manage it at the next checkout; a lost phone uses email meanwhile.

---

## 15. Open questions (Verify before the PR that needs them)

The four customer-account-extension questions from the earlier plan are **resolved — they no longer apply** (no extension, no backend-URL wiring, no `network_access` approval, no `sessionToken.dest`). What remains:

| Question | Needed for |
|---|---|
| QR rendering approach in `qr.server.ts`: vendor a tiny encoder vs. add a reviewed dependency (CLAUDE.md dependency rule) | PR 1 |
| Byte budget: does the one `hasAuthenticator` line fit in `checkout-guard.js` under the 10 KB Theme Check limit? (measure; if not, read it in `checkout-guard-code.js`) | PR 4 |
| Confirm `issuePass` accepts a null `codeId` cleanly everywhere it's called (and the optional `emailVerificationCodeId` migration) | PR 1 / PR 3 |

Reference: [RFC 6238](https://www.rfc-editor.org/rfc/rfc6238) and [RFC 4226](https://www.rfc-editor.org/rfc/rfc4226) for the TOTP/HOTP math; [RFC 4648](https://www.rfc-editor.org/rfc/rfc4648) for base32; the app proxy and `authenticateProxy` work exactly as the existing email-code routes use them.
