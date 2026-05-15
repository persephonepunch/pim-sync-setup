# PIM Sync

A Webflow Designer Extension that synchronizes product catalog data across Shopify, Xano, and Webflow CMS — with asset migration, AI-powered ALT text, WCAG 2.2 AA accessibility, variant galleries, commerce widgets, tag reconciliation, and non-destructive publish to live Shopify.

## Documents

| Document | Description |
|---|---|
| [Functional Spec & Test Plan](FUNCTIONAL-SPEC.md) | Webflow extension requirements, compliance matrix, 58-point regression harness |
| [Shopify App Spec](docs/SHOPIFY-APP-SPEC.md) | PIM Sync Shopify app — OAuth, billing, webhooks, data model, worker architecture |
| [Audit Results](docs/AUDIT-RESULTS.md) | 78-point codebase audit against Shopify App Store requirements |
| [Test Harness](docs/test-harness.html) | Interactive regression test — runs 58 spec coverage checks in-browser |
| [Docs Hub](docs/index.html) | All specs rendered with sidebar navigation and live test runner |
| [Explainer Presentation](docs/video/presentation.html) | 13-slide animated deck for screen recording |
| [Video Script](docs/video/SCRIPT.md) | 4:30 narration script with scene-by-scene visual directions |

## How to Use

### For Strategy / Positioning

This system provides a **unified product information management layer** between Shopify (commerce), Xano (content API), and Webflow (CMS/frontend). Key capabilities:

- **Three-way data mirror** — changes in any system (Shopify Admin, Xano tables, or Webflow CMS) propagate to all others via webhooks and scheduled sync
- **Standalone or paired** — works with just Xano + Webflow, or paired with the [PIM Sync Shopify App](https://apps.shopify.com/pim-sync) for full Admin API integration
- **Asset pipeline** — product images migrate from Shopify CDN through Xano into Cloudflare R2 with immutable caching and automatic format conversion
- **AI-enriched media** — Cloudflare Workers AI generates accessible ALT text and auto-tags from product images
- **Non-destructive publish** — upsert-only mutations, draft-first products, field-level diffs, price confirmation gates

### For Application Submission

The [Functional Spec](FUNCTIONAL-SPEC.md) contains:
- Complete feature surface (13 sections, 812 lines)
- GDPR Article mapping (Art. 5, 17, 25, 28, 30, 32, 44)
- CCPA/CPRA compliance matrix
- PCI DSS compliance statement
- EU Omnibus Directive integration
- WCAG 2.2 AA accessibility requirements with ARIA role table
- 58 regression test cases across 14 categories
- API group isolation documentation (PIM vs CRM boundary)

### For Development / QA

1. Open the [Test Harness](docs/test-harness.html) in a browser
2. Click **Run Spec Audit** to validate the functional spec against 58 checks
3. Click **View Details** for per-section pass/fail breakdown
4. Review any FAIL items and update the spec or codebase accordingly

### Quick Test

```bash
# Serve docs locally
cd docs && python3 -m http.server 8787

# Open test harness
open http://localhost:8787/test-harness.html

# Open full docs hub
open http://localhost:8787/index.html

# Open explainer presentation
open http://localhost:8787/video/presentation.html
```

## Architecture

### Deployment Modes

| Mode | Description |
|---|---|
| **Standalone** | Webflow extension + Xano only. Products managed via CSV/webhook/manual entry. Commerce via Shopify Buy Button SDK. |
| **Paired** | Webflow extension + PIM Sync Shopify App. Full 3-way mirror with OAuth, webhooks, market pricing, translations. |

### System Components

| Component | Platform | Purpose |
|---|---|---|
| Designer Extension | Webflow App (component-sync) | Product browsing, component insertion, embed code generation |
| PIM Backend | Xano (`api:Ksv9J-nS`, workspace 4) | Product data, sync jobs, merchant config |
| Asset CDN | Cloudflare R2 + `asset-mirror-cdn` Worker | Image/video/3D model storage and delivery |
| Webflow Sync | `webflow-sync` Worker | Bidirectional CMS sync with echo detection |
| Publish Sync | `publish-sync` Worker | Translations, metafields, core field mutations (Shopify app) |
| FX Cache | `fx-cache` Worker | ECB/BoC exchange rates for market pricing |
| Translate Proxy | `translate-proxy` Worker | Cloudflare Workers AI for translations and ALT text |
| Omnibus Audit | `omnibus-audit` Worker | EU Omnibus Directive price compliance |
| Shopify App | Cloudflare Pages (`pim-sync.pages.dev`) | OAuth, billing, embedded admin UI |

### Data Flow

```
Shopify Products (Admin API)
    |
    | webhooks (products/create, products/update)
    v
Xano PIM Tables (164, 166, 175, 176, 179)
    |
    | webflow-sync worker
    v
Webflow CMS (Products, Variants, Categories, Tags)
    |
    | Webflow webhooks (collection_item_changed)
    v
Xano (reverse sync) --> Shopify App (publish-sync worker) --> Shopify Admin API
```

### API Group Isolation

| API Group | Owner | Tables | Scope |
|---|---|---|---|
| `api:Ksv9J-nS` (ID 66) | PIM Sync | 164, 166, 175, 176, 179 | Product catalog, sessions, config |
| `api:1Zsx4CNw` | CRM Sync | 180, 181, 182, 183 | Customer identity, consent, claims |

Independent API keys. Zero cross-app access. Key rotation does not affect the other app.

## Compliance

| Standard | Status | Details |
|---|---|---|
| GDPR | Compliant | Art. 5, 17, 25, 28, 30, 32, 44 mapped. No customer PII. Data minimization. 48h erasure on shop/redact. |
| CCPA/CPRA | Compliant | No sale of PI. Service provider agreements with Xano and Cloudflare. |
| PCI DSS | N/A (no card data) | All checkout via Shopify Storefront API. No payment fields rendered. |
| EU Omnibus | Integrated | 30-day price history badge on PDP. Audit worker runs daily. |
| WCAG 2.2 AA | Enforced | ARIA roles on all components. Keyboard navigation. Focus trapping. Audit-before-insert. |
| Section 508 | Satisfied | Via WCAG 2.2 AA conformance. |

## Security

| Layer | Mechanism |
|---|---|
| Transport | TLS 1.2+ everywhere. HSTS on embed responses. |
| Authentication | HMAC-SHA256 on all webhooks. Bearer tokens for Xano. OAuth for Webflow. |
| Input validation | Client-side regex + server-side re-validation. `escapeHtml()` on all DOM insertion. |
| Secrets | Encrypted in Cloudflare KV/env. Independent rotation per service. |
| CORS | Public endpoints are read-only GET. Mutation endpoints require auth. |
| Logging | Every sync job logged. Tokens never logged. No PII in logs. |

## Feature Coverage

| Feature | Spec Section | Test IDs |
|---|---|---|
| Asset migration pipeline | 1 | S01–S03 |
| AI ALT text and auto-tagging | 2 | S04–S06 |
| Accessibility (WCAG 2.2 AA) | 3 | S07–S10 |
| Form validation and regex | 4 | S11–S14 |
| Variant thumbnail slider | 5 | S15–S18 |
| Commerce widgets | 6 | S19–S22 |
| Tag reconciliation | 7 | S23–S25 |
| Three-way data mirror | 8 | S26–S30 |
| Non-destructive publish | 9 | S31–S36 |
| Compliance | 11 | S37–S44 |
| Security | 12 | S45–S55 |
| Meta / structure | — | S56–S58 |

## Related Repositories

| Repository | Description |
|---|---|
| [crm-sync-setup](https://github.com/persephonepunch/crm-sync-setup) | CRM Sync — Auth, Tag System, GA4 Integration, Shopify + Webflow + Xano |
| [PIM Sync Shopify App](https://apps.shopify.com/pim-sync) | Live Shopify App Store listing |

## License

Proprietary. All rights reserved.
