# PIM Sync — App Store Submission Audit

> Automated audit against [shopify-app-checklist.llm.md](https://shopify-checklist.pages.dev/shopify-app-checklist.llm.md) (May 2026).  
> Run date: 2026-05-15

## Summary

| Section | Pass | Fail | Review | Total |
|---------|------|------|--------|-------|
| 1. Dev Dashboard & Config | 10 | 0 | 1 | 11 |
| 2. Authentication & Tokens | 3 | 8 | 3 | 14 |
| 3. Protected Customer Data | 5 | 0 | 1 | 6 |
| 4. Data Security | 3 | 2 | 1 | 6 |
| 5. GDPR / Privacy | 9 | 1 | 0 | 10 |
| 6. App Store Listing | 9 | 0 | 3 | 12 |
| 7. Billing | 3 | 0 | 1 | 4 |
| 8. App Functionality | 4 | 0 | 1 | 5 |
| 9. Webhooks & Sync | 3 | 1 | 1 | 5 |
| 10. Post-Launch | 2 | 1 | 2 | 5 |
| **Totals** | **51** | **13** | **14** | **78** |

**Pass rate: 65% (51/78)**  
**Blockers (FAIL): 13**  
**Manual checks needed (REVIEW): 14**

---

## Critical Failures (Submission Blockers)

### AUTH-01: No immediate OAuth on install/reinstall
- **Reqs:** 2.1, 2.2, 2.4
- **Severity:** Critical (submission rejection)
- **File:** `app/routes/auth.login.jsx:33-60`
- **Issue:** Login page shows a form asking for manual `.myshopify.com` entry. OAuth must initiate immediately with no pre-auth UI.
- **Fix:** Remove the login form. For embedded apps installed from the App Store, Shopify provides the shop context automatically. The `auth.login` route should redirect straight to OAuth without rendering UI.

### AUTH-02: No token refresh implementation
- **Reqs:** 2.6, 2.7, 2.8, 2.9, 2.10, 2.11
- **Severity:** Critical (mandatory since Apr 2026)
- **Files:** `app/services/kv-session-storage.server.ts`, `app/routes/app.tsx`
- **Issues:**
  - Access token TTL tracked as 30-day session TTL, not 60-min token TTL
  - Refresh token stored unencrypted in KV
  - No proactive token refresh (5 min before expiry)
  - No 401 fallback refresh — errors thrown, user sees "Session expired"
  - No monitoring/alerting for refresh failures
- **Fix:** Implement token lifecycle management:
  1. Track `expires_at` per access token (60-min window)
  2. Add middleware that refreshes proactively before expiry
  3. Add 401 catch that triggers refresh before re-throwing
  4. Encrypt refresh tokens at rest in KV
  5. Log refresh failures with structured logging

### WEBHOOK-01: Blocking operations in webhook handler
- **Req:** 9.4
- **Severity:** Critical (webhook delivery failures)
- **File:** `app/routes/api.webhooks.tsx:102-170`
- **Issue:** `COLLECTIONS_UPDATE` handler chains 6-8 `await` calls (Xano config, Worker echo-check, Webflow get/update/publish, Xano sync state) in the critical path. Shopify requires 200 response within 5 seconds.
- **Fix:** Return 200 immediately, dispatch work to a Cloudflare Queue or Durable Object. Pattern:
  ```
  // Acknowledge immediately
  ctx.waitUntil(processWebhook(topic, payload, shop));
  return new Response(null, { status: 200 });
  ```

### GDPR-01: GDPR handler returns 500
- **Req:** 5.6
- **Severity:** High (compliance violation)
- **File:** `app/routes/api.webhooks.tsx:65`
- **Issue:** When Xano backend is unavailable, GDPR handlers return `500`. Shopify requires 200-series responses.
- **Fix:** Return `202 Accepted` and queue the GDPR request for retry. Log the failure for manual follow-up.

### SEC-01: Sensitive data in logs
- **Req:** 4.5
- **Severity:** Medium
- **File:** `app/routes/app.tsx:18`
- **Issue:** `console.log("Auth bounce:", e.status, Object.fromEntries(e.headers.entries()))` logs full response headers which may contain tokens or session data.
- **Fix:** Log only `e.status`, not headers.

### MON-01: No structured monitoring
- **Req:** 10.4
- **Severity:** Medium (post-launch risk)
- **Issue:** Only `console.log` throughout codebase. No error tracking (Sentry/Rollbar), no structured logging, no webhook delivery metrics.
- **Fix:** Add at minimum: Cloudflare Logpush for structured logs, and an error tracking service for uncaught exceptions.

---

## Detailed Audit by Section

### 1. Dev Dashboard & Config (10 PASS / 0 FAIL / 1 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 1.1 | App in Dev Dashboard | PASS | `shopify.app.toml:3` — `client_id = "f870cc29..."` |
| 1.2 | Valid client_id | PASS | Present and correctly formatted |
| 1.3 | application_url clean | PASS | `"https://pim-sync.pages.dev"` — no forbidden words |
| 1.4 | embedded = true | PASS | `shopify.app.toml:6` |
| 1.5 | Minimum scopes | REVIEW | 20 scopes declared — verify each is used |
| 1.6 | No legacy install flow | PASS | `use_legacy_install_flow = false` |
| 1.7 | redirect_urls set | PASS | `["https://pim-sync.pages.dev/auth/callback"]` |
| 1.8 | webhooks.api_version | PASS | `api_version = "2026-04"` |
| 1.9 | Compliance webhooks | PASS | All 3 declared under `[webhooks.privacy_compliance]` |
| 1.10 | CLI-managed extensions | PASS | 2 extensions with `.toml` configs |
| 1.11 | CLI >= 3.84.1 | PASS | Installed: 3.94.3 |

### 2. Authentication & Tokens (3 PASS / 8 FAIL / 3 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 2.1 | Immediate OAuth on install | FAIL | `auth.login.jsx` shows manual shop entry form |
| 2.2 | Immediate OAuth on reinstall | FAIL | Same form blocks reinstall flow |
| 2.3 | Redirect to app UI after OAuth | PASS | `auth.$.jsx` redirects to app |
| 2.4 | No manual shop entry | FAIL | `auth.login.jsx:40-56` has `<input name="shop">` |
| 2.5 | expiring=1 parameter | REVIEW | `expiringOfflineAccessTokens: true` set — library should handle |
| 2.6 | Access token TTL tracking | FAIL | Session TTL is 30 days, not 60-min token TTL |
| 2.7 | Refresh token encrypted | FAIL | Stored unencrypted in KV |
| 2.8 | Proactive token refresh | FAIL | No refresh logic found |
| 2.9 | 401 fallback refresh | FAIL | Error thrown, user sees "Session expired" reload button |
| 2.10 | Update both tokens on refresh | FAIL | No refresh logic exists |
| 2.11 | Refresh failure monitoring | FAIL | No monitoring infrastructure |
| 2.12 | No eternal token assumption | PASS | `expiringOfflineAccessTokens: true` enabled |
| 2.13 | Session tokens for embedded auth | PASS | `AppProvider embedded` with server-side sessions |
| 2.14 | Chrome incognito support | REVIEW | Architecture compatible, but needs manual test |

### 3. Protected Customer Data (5 PASS / 0 FAIL / 1 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 3.1 | Access level determined | PASS | Level 0 — no customer scopes |
| 3.2 | Access requested | PASS | N/A — Level 0 |
| 3.3 | Specific field access | PASS | N/A — Level 0 |
| 3.4 | Null handling for redacted fields | PASS | N/A — no customer fields queried |
| 3.5 | Data protection details | PASS | `privacy.tsx` covers storage, HTTPS, data practices |
| 3.6 | Tested on non-dev store | REVIEW | Cannot verify from code |

### 4. Data Security (3 PASS / 2 FAIL / 1 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 4.1 | Tokens encrypted at rest | FAIL | `.env` plaintext (gitignored but local dev risk) |
| 4.2 | No secrets in client code | PASS | Only `SHOPIFY_API_KEY` (intentionally public) |
| 4.3 | HTTPS everywhere | PASS | All external calls use HTTPS |
| 4.4 | Webhook HMAC validation | PASS | `shopify.authenticate.webhook(request)` |
| 4.5 | No PII/tokens in logs | FAIL | `app.tsx:18` logs full response headers |
| 4.6 | Platform secrets manager | REVIEW | `wrangler.toml` configured but verify secrets provisioned |

### 5. GDPR / Privacy (9 PASS / 1 FAIL / 0 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 5.1 | customers/data_request handler | PASS | `api.webhooks.tsx:202` |
| 5.2 | customers/redact handler | PASS | `api.webhooks.tsx:206` |
| 5.3 | shop/redact handler | PASS | `api.webhooks.tsx:210` with `purgeShopData` |
| 5.4 | POST + JSON accepted | PASS | Framework handles deserialization |
| 5.5 | HMAC validated | PASS | SDK's `authenticate.webhook()` |
| 5.6 | 200-series response | FAIL | Returns 500 when backend unavailable (`api.webhooks.tsx:65`) |
| 5.7 | Privacy policy URL | PASS | `/privacy` route, public, with contact info |
| 5.8 | Policy content complete | PASS | Covers: data collected, usage, retention, storage, contact |
| 5.9 | Minimal data collection | PASS | Product data only, no customer PII |
| 5.10 | Data deleted when unneeded | PASS | `purgeShopData` on uninstall + 30-day session TTL |

### 6. App Store Listing (9 PASS / 0 FAIL / 3 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 6.1 | App name unique | PASS | "PIM Sync" — no "Shopify" |
| 6.2 | Icon 1200x1200 | PASS | `public/pim-1200.jpg` verified |
| 6.3 | Subtitle written | REVIEW | Partner Dashboard setting |
| 6.4 | App details | REVIEW | Partner Dashboard setting |
| 6.5 | Screenshots 1600x900 | PASS | 16+ files in `/public/images/intro/` |
| 6.6 | No Shopify trademarks | PASS | No trademark usage in assets |
| 6.7 | No testimonials | PASS | No testimonial content |
| 6.8 | Demo screencast | PASS | `/images/intro/pim-sync-intro.mp4` in demo route |
| 6.9 | Test credentials | PASS | `.env.example` provided |
| 6.10 | Emergency contact | PASS | `ysl@story-story.ai` in multiple routes |
| 6.11 | Email clean | PASS | No "Shopify" in email |
| 6.12 | Category set | REVIEW | Partner Dashboard setting |

### 7. Billing (3 PASS / 0 FAIL / 1 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 7.1 | Shopify Billing API | PASS | `appSubscriptionCreate` mutation in `app.billing.tsx` |
| 7.2 | test: true for dev | PASS | `test: isDevStore` (line 64) |
| 7.3 | test: false for production | REVIEW | Logic correct, verify at submission |
| 7.4 | Upgrade without reinstall | PASS | "Switch" UI for existing subscribers |

### 8. App Functionality (4 PASS / 0 FAIL / 1 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 8.1-8.7 | Platform rules | PASS | Not checkout/theme; unique; factual |
| 8.8 | Performance | PASS | Cloudflare Pages CDN, KV sessions |
| 8.9 | Rate limit handling | PASS | `webflow.server.ts:31-36` — 429 + Retry-After + backoff |
| 8.10 | Idempotent webhooks | REVIEW | Verify Xano backend idempotency |

### 9. Webhooks & Sync (3 PASS / 1 FAIL / 1 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 9.1 | GraphQL registration | PASS | TOML config auto-registered by CLI |
| 9.2 | HMAC verified | PASS | SDK `authenticate.webhook()` |
| 9.3 | Tolerates unknown fields | PASS | `payload as any`, optional chaining |
| 9.4 | 200 within 5s, async work | FAIL | 6-8 blocking `await` calls in handler |
| 9.5 | Subrequest limits | REVIEW | ~8-9 per item, verify under load |

### 10. Post-Launch (2 PASS / 1 FAIL / 2 REVIEW)

| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 10.1 | API version current | PASS | `2026-04` — current |
| 10.2 | Test credentials current | REVIEW | Operational — verify on schedule |
| 10.3 | App matches listing | REVIEW | Verify visuals/pricing in Partner Dashboard |
| 10.4 | Monitoring active | FAIL | No structured logging or error tracking |
| 10.5 | Scope changes via CLI | PASS | Process documented |

---

## Priority Fix Order

### P0 — Submission Blockers (fix before review)
1. **AUTH-01** — Remove manual shop entry form from `auth.login.jsx`
2. **AUTH-02** — Implement token refresh lifecycle (proactive + 401 fallback)
3. **WEBHOOK-01** — Move webhook work to background (`ctx.waitUntil` or queue)
4. **GDPR-01** — Change GDPR error response from 500 to 202

### P1 — High Priority (fix before launch)
5. **SEC-01** — Remove header logging from `app.tsx:18`
6. **MON-01** — Add structured logging + error tracking
7. Verify Cloudflare Pages secrets are provisioned (not just `.env`)
8. Encrypt refresh tokens at rest in KV

### P1.5 — Xano API Group Isolation (new)
9. **XANO-01** — Create dedicated PIM API group in Xano (workspace 4, tables 164/166/175/176/179)
10. **XANO-02** — Generate PIM-scoped API key for the new group
11. **XANO-03** — Update `XANO_BASE_URL` in PIM Sync app + all 6 workers to new group endpoint
12. **XANO-04** — Update `XANO_API_KEY` via `wrangler pages secret put` + worker secrets
13. **XANO-05** — Verify PIM key cannot access CRM tables (180–183) — expect 401/403
14. **XANO-06** — Verify CRM key (`api:1Zsx4CNw`) cannot access PIM tables (164–179) — expect 401/403

### P2 — Review Items (verify manually)
15. Confirm all 20 scopes are justified
16. Test full OAuth flow in Chrome incognito
17. Verify Xano backend idempotency for webhook handlers
18. Confirm `test: false` in production billing
19. Test on non-development store

---

## Xano Isolation Test Plan

### Pre-migration (current state)

Both apps share `api:1Zsx4CNw` — **no isolation**. Either app can read/write any table.

### Post-migration (target state)

| Test ID | Test | Method | Expected |
|---------|------|--------|----------|
| ISO-01 | PIM key reads table 164 (Products) | `GET /workspace/4/table/164/content` | 200 OK |
| ISO-02 | PIM key reads table 166 (Images) | `GET /workspace/4/table/166/content` | 200 OK |
| ISO-03 | PIM key reads table 175 (Sessions) | `GET /workspace/4/table/175/content` | 200 OK |
| ISO-04 | PIM key reads table 176 (Config) | `GET /workspace/4/table/176/content` | 200 OK |
| ISO-05 | PIM key reads table 179 (Sync State) | `GET /workspace/4/table/179/content` | 200 OK |
| ISO-06 | PIM key reads table 180 (Users) | `GET /workspace/4/table/180/content` | **401/403 — blocked** |
| ISO-07 | PIM key reads table 181 (Claims) | `GET /workspace/4/table/181/content` | **401/403 — blocked** |
| ISO-08 | PIM key reads table 182 (Extras) | `GET /workspace/4/table/182/content` | **401/403 — blocked** |
| ISO-09 | PIM key reads table 183 (Consent) | `GET /workspace/4/table/183/content` | **401/403 — blocked** |
| ISO-10 | CRM key reads table 180 (Users) | `GET /workspace/4/table/180/content` | 200 OK |
| ISO-11 | CRM key reads table 164 (Products) | `GET /workspace/4/table/164/content` | **401/403 — blocked** |
| ISO-12 | PIM app full sync after migration | Trigger full sync from Shopify admin | Products sync to Xano + Webflow |
| ISO-13 | CRM app auth after migration | Login via email/password | JWT issued, user in Xano |
| ISO-14 | PIM workers use new endpoint | Check each worker's `XANO_BASE_URL` | Points to PIM group, not `api:1Zsx4CNw` |

---

## Key Dates

| Date | Requirement |
|------|-------------|
| Feb 2025 | Scopes reviewed on every submission |
| Dec 2025 | Protected customer data scopes enforced for web pixels |
| **Apr 1, 2026** | **Expiring offline tokens mandatory** (AUTH-02 is critical) |
| Mar 2026 | RBAC for partner orgs; clearer image standards |
| Apr 2026 | New submission experience with AI self-review |
