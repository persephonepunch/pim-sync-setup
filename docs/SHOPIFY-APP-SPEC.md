# PIM Sync for UCP Data — Functional Specification

> Functional specification for marketplace submission readiness and LLM audit. Pairs with [shopify-checklist.pages.dev](https://shopify-checklist.pages.dev/).

**App name:** PIM Sync for UCP Data  
**Developer:** ysldev  
**Version:** 1.0.0  
**Status:** Live — [apps.shopify.com/pim-sync](https://apps.shopify.com/pim-sync)  
**API version:** 2026-04  
**Runtime:** Cloudflare Pages (advanced mode)  
**Last updated:** May 2026

---

## 1. Product & Scope Definition

### 1.1 What It Does

PIM Sync replaces CSV-based product management with structured GraphQL automation built on the Universal Commerce Protocol. It expands a single-market Shopify catalog into localized EU/EMEA markets — handling product mirroring, currency conversion, AI translations, Omnibus compliance pricing, and Webflow CMS synchronization from one embedded dashboard.

### 1.2 Target User

Shopify merchants selling internationally who need localized product catalogs across multiple markets and want headless Webflow storefronts kept in sync with Shopify.

### 1.3 Shopify API Scopes

| Scope | Justification |
|-------|---------------|
| `read_products`, `write_products` | Mirror products into market variants, update metafields |
| `read_markets`, `write_markets` | Configure target markets (EU, UK, EMEA, Americas, APAC) |
| `read_translations`, `write_translations` | Register translated content to Global Catalog |
| `read_locales`, `write_locales` | Query shop locales for translation targets |
| `read_price_rules`, `write_price_rules` | Omnibus compliance price history tracking |
| `read_inventory`, `write_inventory` | Variant inventory sync across markets |
| `read_files`, `write_files` | Image and 3D asset management |
| `read_publications`, `write_publications` | Market-specific publication triggers |
| `read_metaobjects`, `write_metaobjects` | Structured product data storage |
| `read_metaobject_definitions`, `write_metaobject_definitions` | Custom taxonomy definitions |

### 1.4 Protected Customer Data

- **Access level:** 0 (no customer PII accessed)
- The app does not read, store, or process customer personal data
- All data operations are product/catalog scoped

---

## 2. Architecture

### 2.1 System Components

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | React Router v7 + Shopify Polaris 12 | Embedded admin UI |
| Runtime | Cloudflare Pages (_worker.js) | SSR + API routes |
| Sessions | Cloudflare KV (`SESSIONS` namespace) | OAuth token storage |
| PIM Backend | Xano — **PIM API group** (dedicated) | Product data, sync jobs, config |
| Asset Storage | Cloudflare R2 (`prod-catalog` bucket) | Images and 3D models |
| Edge Workers | 6 Cloudflare Workers | CDN, FX, translate, publish, Webflow sync, audit |
| CMS | Webflow REST API v2 | Collection sync |
| Shopify API | Admin GraphQL v2026-04 | Products, markets, translations |

### 2.2 Worker Architecture

| Worker | Function | Trigger |
|--------|----------|---------|
| `asset-mirror-cdn` | Serves R2 images with CORS, caching, range requests | HTTP request |
| `fx-cache` | Caches FX rates, triggers price recalculation | Scheduled (cron) |
| `translate-proxy` | Proxies to Cloudflare Workers AI (Meta m2m100) | HTTP request |
| `publish-sync` | Publishes market variants to Shopify Global Catalog | HTTP request |
| `webflow-sync` | Bidirectional Webflow CMS sync | HTTP request / webhook |
| `omnibus-audit` | Scheduled EU Omnibus compliance price audit | Scheduled (cron) |

### 2.3 Data Flow

```
Shopify Products (Admin API)
    ↓ full_sync
Xano PIM (Tables 164, 166)
    ↓ asset_mirror
Cloudflare R2 (prod-catalog/{region}/{sku}/)
    ↓ webflow_sync
Webflow CMS (Collections: Products, Variants, Categories, Tags)
```

---

## 3. Authentication & Sessions

### 3.1 OAuth Flow

| Step | Implementation |
|------|---------------|
| Install | OAuth initiates immediately — no pre-auth UI |
| Reinstall | Same immediate OAuth — merchant data preserved |
| Token exchange | `expiring=1` sent (mandatory since Apr 2026) |
| Access token | `shpua_` prefix, 60-min TTL, stored in Cloudflare KV |
| Refresh token | `shprt_` prefix, 90-day TTL, stored in KV |
| Refresh strategy | Proactive refresh 5 min before expiry |
| Fallback | 401 triggers token refresh; new OAuth if refresh token expired |
| Session tokens | Used for embedded auth (not cookies/localStorage) |

### 3.2 Session Storage

- **Primary:** Cloudflare KV namespace `SESSIONS`
- **Fallback:** In-memory Map (development only)
- **Methods:** `storeSession`, `loadSession`, `deleteSession`, `deleteSessions`, `findSessionsByShop`
- **Merchant config:** Persisted to Xano Table 175 (shop + tokens + Webflow config)

### 3.3 Webflow OAuth

- Separate OAuth flow for Webflow API access
- Redirect callback at `/app/webflow/callback`
- Token stored in Xano merchant config (Table 176)
- Scopes: site read/write, CMS read/write

---

## 4. Core Features

### 4.1 Dashboard (`app._index`)

| Element | Data Source | Behavior |
|---------|-------------|----------|
| Pipeline health cards | Xano, Cloudflare, Webflow APIs | Green/red status per service |
| Translation queue summary | Xano translations table | Pending/translated/reviewed/published counts |
| Plan status | Shopify Billing API | Current plan, usage, limits |
| Recent sync jobs | Xano sync jobs table | Last 5 jobs with status |

### 4.2 Product Catalog (`app.products`)

| Feature | Specification |
|---------|--------------|
| List view | 50 products per page, cursor pagination |
| Search | By title, handle, or SKU |
| Columns | Title, handle, image count (Xano), sync status, price, variants |
| Product detail | Variant list, market pricing, metafields, image gallery |
| Actions | Trigger sync, translate, publish per product |

### 4.3 Markets Management (`app.markets`)

| Feature | Specification |
|---------|--------------|
| Supported markets | US, CA, EU, UK, EMEA (expandable to Americas, APAC) |
| Supported locales | en-US, en-CA, fr-CA, en-GB, fr-FR, de-DE, es-ES, it-IT, nl-NL, pt-PT, pl-PL, sv-SE |
| Configure | Enable/disable markets, assign locales, set currencies |
| FX conversion | Automatic rate fetch for English-speaking markets |
| Price rounding | Configurable: .99, .95, .50 |

### 4.4 Translation Queue (`app.translations`)

| Stage | Description |
|-------|-------------|
| Pending | Product queued for translation |
| Translated | AI model output received (Meta m2m100-1.2B via Workers AI) |
| Reviewed | Human review completed (4-stage team workflow) |
| Published | Registered to Shopify via `translationsRegister` mutation |

**Translation targets:** Title, description, alt text per locale  
**Model:** Meta m2m100-1.2B (via Cloudflare Workers AI)  
**Languages:** French, German, Dutch, Spanish, Italian, Portuguese, Polish, Swedish, Brazilian Portuguese

### 4.5 Webflow Integration (`app.webflow`)

| Feature | Specification |
|---------|--------------|
| Connection | OAuth to Webflow API |
| Site selection | Pick target Webflow site |
| Collection mapping | Products, Variants, Categories, Tags, Google Product Categories |
| Auto-scaffold | Create missing collections/fields in Webflow |
| Field mapping | Configurable metafield → CMS field mapping |
| Batch size | 100–1000 products per sync run |
| Conflict detection | Side-by-side Shopify vs Webflow comparison |
| Resolution | Keep-shopify or keep-webflow per field |
| Sync directions | `shopify_to_xano`, `xano_to_webflow`, `webflow_to_xano` |

### 4.6 Omnibus Compliance (`app.compliance`)

| Feature | Specification |
|---------|--------------|
| Scope | EU Omnibus Directive (2019/2161) |
| Audit | Detect price increases within 30-day window |
| Price history | Per-market variant with timestamps |
| Display | Current vs historical pricing with VAT breakdowns |
| Override | Manual price changes with full audit trail |
| Worker | `omnibus-audit` runs on scheduled cron |

### 4.7 Sync Management (`app.sync`)

| Sync Type | Direction | Description |
|-----------|-----------|-------------|
| Full Catalog | Shopify → Xano | Import all products, variants, images |
| Price Sync | FX API → Xano → Shopify | Recalculate market prices |
| Translate | Xano → Workers AI → Xano | Batch translate queued items |
| Asset Mirror | Shopify → R2 | Copy images to Cloudflare R2 CDN |
| Publish | Xano → Shopify | Push market variants to Global Catalog |
| Webflow Sync | Xano → Webflow | Push products to Webflow CMS |

**Job tracking:** Each sync creates a job record with:
- `id`, `shop`, `job_type`, `status`, `direction`
- `product_count`, `error_message`
- `started_at`, `completed_at`, `created_at`

**Statuses:** `pending` → `running` → `completed` | `failed`

### 4.8 Image & Asset Pipeline

| Feature | Specification |
|---------|--------------|
| Ingest | Product images from Shopify Admin API |
| Storage | Cloudflare R2 bucket `prod-catalog` |
| Path format | `assets/{region}/{sku}/{filename}` |
| CDN | `asset-mirror-cdn` worker with CORS, caching, range requests |
| 3D models | GLB, USDZ, STL, OBJ, 3MF (Enterprise tier) |
| Alt text | Per-locale translation via translation queue |
| Push | On-demand push to Shopify CDN |

### 4.9 Settings (`app.settings`)

| Setting | Tier | Description |
|---------|------|-------------|
| Source language | All | Default locale for source content (en-US, en-CA, fr-CA) |
| Price rounding | All | .99, .95, .50 |
| Target markets | All | Enable/disable market regions |
| Xano credentials | Plus+ | Custom Xano base URL + API key |
| Cloudflare account | Plus+ | Custom CF account ID |
| Webflow token | Pro+ | OAuth-managed, stored in Xano |

### 4.10 Testing Suite (`app.tests`)

| Validation | What It Checks |
|------------|----------------|
| Xano health | API reachability, table access |
| Webflow collections | Collection exists, fields match schema |
| GPC categories | Google Product Categories taxonomy valid |
| Shopify connection | Admin API auth, scope verification |

---

## 5. Billing & Plans

### 5.1 Plan Matrix

| Plan | Price | Trial | Products | Markets | Key Gates |
|------|-------|-------|----------|---------|-----------|
| Free | $0 | — | 50 | 2 | Basic sync, no Webflow |
| Pro | $69/mo | 30 days | Unlimited | 5 | Webflow sync, translations |
| Plus | $325/mo | 30 days | Unlimited | Unlimited | Background sync, priority, custom infra |
| 3D Enterprise | $525/mo | — | Unlimited | Unlimited | Full 3D pipeline (GLB/USDZ/STL/OBJ/3MF) |
| Non-Profit | $29/mo | 30 days | 500 | 5 | 501(c)(3) verification required |

### 5.2 Feature Gates

| Feature | Free | Pro | Plus | 3D |
|---------|------|-----|------|-----|
| `maxProducts` | 50 | ∞ | ∞ | ∞ |
| `maxMarkets` | 2 | 5 | ∞ | ∞ |
| `webflowSync` | — | Yes | Yes | Yes |
| `backgroundTasks` | — | — | Yes | Yes |
| `omnibusCompliance` | Yes | Yes | Yes | Yes |
| `prioritySync` | — | — | Yes | Yes |
| `model3dSync` | — | — | — | Yes |

### 5.3 Billing Implementation

- All charges via Shopify Billing API (`appSubscriptionCreate` mutation)
- Plan upgrades/downgrades without reinstall
- `test: true` for development; `test: false` for production
- Usage-based billing not currently implemented (flat monthly)

---

## 6. Webhooks

### 6.1 Registered Webhooks

| Topic | Handler Behavior |
|-------|-----------------|
| `products/create` | Queue product for Xano ingestion |
| `products/update` | Update Xano record, flag Webflow conflict if diverged |
| `products/delete` | Mark Xano record deleted, remove from Webflow |
| `collections/create` | Sync collection to Xano taxonomy |
| `collections/update` | Update taxonomy mapping |
| `collections/delete` | Remove taxonomy entry |
| `app_subscriptions/update` | Update merchant plan status |

### 6.2 GDPR Compliance Webhooks

| Topic | Handler Behavior |
|-------|-----------------|
| `customers/data_request` | Return 200 (no customer data stored) |
| `customers/redact` | Return 200 (no customer data to redact) |
| `shop/redact` | Delete merchant config, sync jobs, Xano records within 48h |

### 6.3 Webhook Requirements

- All handlers validate HMAC signature (401 on failure)
- All handlers respond 200 within 5 seconds
- Heavy work dispatched to Workers asynchronously
- Handlers tolerate unknown fields (forward-compatible)
- Handlers are idempotent (duplicate delivery safe)

---

## 7. Data Model (Xano)

### 7.1 API Group Isolation

PIM Sync and CRM Sync share the same Xano instance (`xerb-qpd6-hd8t.n7.xano.io`, workspace 4) but **must** use separate API groups with independent API keys.

| API Group | Owner | Tables | API Key Env Var |
|-----------|-------|--------|-----------------|
| **PIM group** (`api:Ksv9J-nS`, ID 66) | PIM Sync | 164, 166, 175, 176, 179 | `XANO_API_KEY` (PIM-scoped) |
| **CRM group** (`api:1Zsx4CNw`) | CRM Sync | 180, 181, 182, 183 | `XANO_API_KEY` (CRM-scoped) |

**Why separate groups:**
- CRM Sync is the core Webflow app, registered first — keeps existing `api:1Zsx4CNw`
- PIM Sync is a Webflow addition to the live Shopify app — gets a new dedicated API group
- Separate API keys prevent cross-app data access (PIM cannot read user tables, CRM cannot read product tables)
- Independent key rotation — revoking one key does not break the other app

**Migration steps:**
1. ~~Create new API group in Xano (workspace 4) scoped to tables 164, 166, 175, 176, 179~~ **Done** — `api:Ksv9J-nS` (ID 66), created 2026-05-15
2. Generate a new API key for the PIM group (Xano dashboard → Workspace 4 → API group 66)
3. Update `XANO_BASE_URL` in PIM Sync app and all PIM workers to the new group endpoint
4. Update `XANO_API_KEY` in PIM Sync env (Cloudflare Pages secrets + worker secrets)
5. Verify PIM Sync cannot access tables 180–183 (CRM tables)
6. Verify CRM Sync cannot access tables 164–179 (PIM tables)

### 7.2 PIM Tables (API Group: PIM)

| Table ID | Name | Key Fields |
|----------|------|------------|
| 164 | Products | id, handle, sku, title, description, vendor, product_type, tags, status |
| 166 | Product Images | product_id, src, src_r2, alt, position, locale_alts |
| 175 | Sessions | shop, access_token, refresh_token, expires_at, webflow_api_token |
| 176 | Merchant Config | shop, target_markets[], source_lang, pricing_rounding, xano_base_url, xano_api_key, cf_account_id, webflow_site_id, webflow_collection_ids |
| 179 | Sync State | shopify_gid, xano_id, webflow_item_id, last_synced_at, status |

### 7.3 CRM Tables (API Group: CRM — `api:1Zsx4CNw`)

| Table ID | Name | Owner |
|----------|------|-------|
| 180 | Storefront Users | CRM Sync only |
| 181 | User Claims | CRM Sync only |
| 182 | User Extras | CRM Sync only |
| 183 | Consent Records | CRM Sync only |

PIM Sync code **must not** reference tables 180–183. CRM Sync code **must not** reference tables 164–179.

### 7.2 Market Variant Schema

```
id: string
product_master_id: string
market: "US" | "CA" | "EU" | "UK" | "EMEA"
locale: Locale
price_local: number
currency: string
omnibus_price: number | null
omnibus_valid_from: string | null
omnibus_valid_through: string | null
title_translated: string
description_translated: string
translation_status: "pending" | "translated" | "reviewed" | "published"
translation_model: string
cf_asset_prefix: string
```

### 7.3 Sync Job Schema

```
id: string
shop: string
job_type: "full_sync" | "price_sync" | "translate" | "asset_mirror" | "publish" | "webflow_sync"
status: "pending" | "running" | "completed" | "failed"
direction: "shopify_to_xano" | "xano_to_shopify" | "xano_to_webflow" | "webflow_to_xano"
product_count: number
error_message: string | null
started_at: timestamp
completed_at: timestamp
created_at: timestamp
```

---

## 8. Extensions

### 8.1 Webflow Designer Extension (`component-sync/`)

| Feature | Description |
|---------|-------------|
| Fetch products | Query Xano/Shopify from Webflow Designer |
| Set content | Apply product text/images to selected elements |
| Insert cards | Add product cards directly to canvas |
| Register components | Create reusable product components |
| Copy embed code | Header, grid, PDP section snippets |
| Configuration | Xano URL, store domain, access token |

### 8.2 Webflow Embeds (`webflow-embeds/`)

23 pre-built HTML/CSS embed templates:
- Collection grid
- Product detail page (PDP)
- Footer, header, navigation
- Image slider
- Keyboard hero (GSAP animation)
- Breadcrumb navigation

### 8.3 3D Download Block (`3d-download-block/`)

- Shopify theme app extension
- Renders download buttons for 3D model files (STL, OBJ, 3MF)
- Files served from Cloudflare R2

---

## 9. Environment & Configuration

### 9.1 Required Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `SHOPIFY_API_KEY` | Shopify Dev Dashboard | App identification |
| `SHOPIFY_API_SECRET` | Shopify Dev Dashboard | OAuth + HMAC validation |
| `SCOPES` | shopify.app.toml | OAuth scope string |
| `SHOPIFY_APP_URL` | Cloudflare Pages URL | OAuth redirect base |
| `XANO_BASE_URL` | Xano PIM API group (`api:Ksv9J-nS`) | `https://xerb-qpd6-hd8t.n7.xano.io/api:Ksv9J-nS` |
| `XANO_API_KEY` | Xano PIM API group | PIM-scoped key (cannot access CRM tables 180–183) |
| `XANO_WORKSPACE_ID` | Xano (default: "4") | Workspace selector |
| `XANO_SESSIONS_TABLE_ID` | Xano (default: "175") | Session table |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare Dashboard | Worker/R2 access |
| `CLOUDFLARE_API_TOKEN` | Cloudflare Dashboard | Worker deployment |
| `CLOUDFLARE_R2_ACCESS_KEY_ID` | Cloudflare R2 | Asset storage auth |
| `CLOUDFLARE_R2_SECRET_ACCESS_KEY` | Cloudflare R2 | Asset storage auth |
| `CLOUDFLARE_R2_BUCKET` | Default: "prod-catalog" | Asset bucket name |
| `CLOUDFLARE_R2_PUBLIC_URL` | Cloudflare R2 | Public asset CDN |
| `WEBFLOW_CLIENT_ID` | Webflow Developer | OAuth client |
| `WEBFLOW_CLIENT_SECRET` | Webflow Developer | OAuth secret |

### 9.2 Cloudflare Bindings

| Binding | Type | Purpose |
|---------|------|---------|
| `SESSIONS` | KV Namespace | OAuth session storage |
| `R2_BUCKET` | R2 Bucket | Product image/asset storage |
| `AI` | Workers AI | Translation model inference |

---

## 10. Security & Data Handling

### 10.1 Token Security

- Access tokens encrypted in Cloudflare KV (not plaintext config)
- No secrets in client-side bundles
- All API calls over HTTPS/TLS
- Webhook HMAC validation on every payload
- No PII in application logs

### 10.2 Data Minimization

- No customer personal data collected or stored
- Product data only (titles, descriptions, prices, images)
- Merchant config limited to: shop domain, API tokens, preferences
- `shop/redact` handler purges all merchant data within 48h of uninstall

### 10.3 Third-Party Data Sharing

| Recipient | Data Shared | Purpose |
|-----------|-------------|---------|
| Xano | Product catalog, merchant config | PIM storage and sync orchestration |
| Cloudflare R2 | Product images, 3D models | CDN and asset delivery |
| Cloudflare Workers AI | Product text (no PII) | AI translation |
| Webflow | Product catalog subset | CMS synchronization |

---

## 11. Installation & Onboarding

### 11.1 Install Flow

1. Merchant clicks "Install" on Shopify App Store
2. OAuth initiates immediately (no pre-auth screen)
3. Merchant approves scopes
4. Redirect to app dashboard inside Shopify Admin
5. Onboarding intro (9-section feature walkthrough)
6. Pipeline health check runs automatically

### 11.2 Reinstallation

- OAuth re-initiates immediately
- Existing merchant config preserved in Xano
- Sync jobs and product data retained
- Webflow connection requires re-authorization

### 11.3 Uninstall

- `app/uninstalled` webhook fires
- `shop/redact` received within 48h
- Handler deletes: Xano merchant config, sync jobs, session data
- R2 assets retained (no customer data) until manual cleanup
- No orphaned webhooks or scripts

---

## 12. Performance & Limits

### 12.1 Rate Limit Handling

| API | Strategy |
|-----|----------|
| Shopify Admin GraphQL | Respect `throttledStatus`, exponential backoff on 429 |
| Xano | 100 req/sec default; batch operations where possible |
| Webflow | 60 req/min; chunked sync with delay |
| Cloudflare Workers | 50 subrequests per invocation; fan-out to queues |

### 12.2 Sync Performance

| Operation | Typical Duration | Batch Size |
|-----------|-----------------|------------|
| Full catalog (500 products) | 2–5 minutes | 50 products/batch |
| Price sync | 30–60 seconds | All variants |
| Webflow sync | 1–3 minutes | 100–1000 items |
| Translation batch | 1–2 minutes | 50 items |
| Asset mirror | 5–10 minutes | 10 concurrent uploads |

### 12.3 Fallback Protection

- 15-minute sync timeout with automatic job failure
- Retry with backoff for transient failures (network, 429, 503)
- No partial writes — sync jobs are atomic per batch
- Conflict detection prevents silent data overwrites

---

## 13. App Store Listing Requirements

### 13.1 Listing Assets

| Asset | Spec | Status |
|-------|------|--------|
| App name | "PIM Sync for UCP Data" — no "Shopify" | Done |
| App icon | 1200x1200px PNG | Done |
| Screenshots | 1600x900px (16:9), 3–6 showing actual UI | Done |
| Demo screencast | English, shows onboarding + sync flow | Done |
| Privacy policy | Hosted URL covering data practices | Done |
| Category | Product management / Inventory | Done |
| App Store URL | [apps.shopify.com/pim-sync](https://apps.shopify.com/pim-sync) | **Live** |

### 13.2 Test Credentials for Review

| Credential | Value |
|------------|-------|
| Dev store URL | Provided to reviewer |
| Xano workspace | Pre-configured (no reviewer setup) |
| Webflow test site | Connected with sample collections |
| Plan | Pro tier enabled for full feature access |

### 13.3 Demo Store

- Pre-populated with 50+ products across 3 markets
- Active Webflow sync demonstrating conflict resolution
- Translation queue with items in all 4 stages
- Omnibus compliance data showing 30-day price history

---

## 14. Post-Launch Maintenance

### 14.1 Monitoring

| Signal | Source | Alert Threshold |
|--------|--------|-----------------|
| Webhook delivery rate | Shopify Dev Dashboard | < 95% |
| Sync job failure rate | Xano logs | > 5% in 1h |
| Worker error rate | Cloudflare Dashboard | > 1% |
| API version deprecation | Shopify changelog | 90 days before sunset |
| Token refresh failures | Application logs | Any failure |

### 14.2 API Version Policy

- Current: `2026-04`
- Migration plan: Update within 30 days of new stable release
- Test on dev store before production migration
- `webhooks.api_version` in TOML kept in sync

---

## Appendix: Route Map

| Route | Page | Plan Gate |
|-------|------|-----------|
| `app._index` | Dashboard | All |
| `app.products` | Product catalog | All |
| `app.products.$id` | Product detail | All |
| `app.markets` | Markets management | All |
| `app.translations` | Translation queue | Pro+ |
| `app.webflow` | Webflow integration | Pro+ |
| `app.webflow.callback` | Webflow OAuth callback | Pro+ |
| `app.webflow.conflicts` | Conflict resolution | Pro+ |
| `app.sync` | Sync management | All |
| `app.compliance` | Omnibus compliance | All |
| `app.billing` | Plan selection | All |
| `app.settings` | Configuration | All |
| `app.tests` | Validation suite | All |
| `api.webhooks` | Webhook handler | — |
| `demo` | Public demo page | — |
| `intro` | Onboarding walkthrough | All |
| `pricing` | Public pricing | — |
| `privacy` | Privacy policy | — |
