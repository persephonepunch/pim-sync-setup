# PIM Sync — Webflow Designer Extension Functional Specification

> Functional specification for the PIM Sync Webflow Designer Extension. Covers the full feature surface: asset migration, NLP-enriched media, accessibility, form validation, variant previews, commerce widgets, tag reconciliation, 3-way data mirroring, and non-destructive publish.

**Extension name:** PIM Sync (component-sync)  
**Webflow App Client ID:** `f91764136187ce75a843bdf2b0b212e273f7bba375907cf51d9a5e5fe9a6bb1e`  
**Xano API Group:** `api:Ksv9J-nS` (ID 66, workspace 4)  
**Workers:** `webflow-sync`, `asset-mirror-cdn`, `publish-sync`, `translate-proxy`  
**Companion Shopify App:** [PIM Sync for UCP Data](https://apps.shopify.com/pim-sync) (optional)  
**Last updated:** May 2026

### Deployment Modes

| Mode | Description | Data Source | Commerce |
|------|-------------|------------|----------|
| **Standalone** | Webflow extension installed without the Shopify app. Merchants connect directly to Xano via API URL + key. | Xano tables only. Products imported via CSV, webhook, or manual entry in Xano. | Shopify Storefront API via Buy Button SDK (merchant provides Storefront Access Token). |
| **Paired** | Webflow extension + PIM Sync Shopify app installed. OAuth tokens and merchant config shared via Xano. | Full 3-way mirror: Shopify Admin API ↔ Xano ↔ Webflow CMS. | Full Shopify checkout integration with real-time inventory, market pricing, and cart persistence. |

**Standalone limitations:**
- No Shopify webhook sync (products/create, products/update) — Xano is the sole source of truth
- No Admin API mutations — publish and reverse sync features are disabled
- No market pricing or FX conversion — single-currency only
- No translation pipeline — ALT text and tags still work via Workers AI
- Cart checkout redirects to a manually configured Shopify store URL rather than an authenticated checkout

**Pairing benefits:**
- OAuth token auto-populated from Shopify app session (table 175)
- Bidirectional webhook sync keeps all three systems in lock-step
- Full publish pipeline handled by the Shopify app's existing workers (`publish-sync`, `fx-cache`, `omnibus-audit`)
- Market pricing with FX conversion across all enabled markets
- Omnibus compliance auditing and price history
- Extension never calls Shopify Admin API directly — it writes to Xano, and the Shopify app owns all Admin API mutations

---

## 1. Asset Migration Pipeline

### 1.1 Shopify → Xano → R2 → Webflow

Products in Shopify carry images hosted on Shopify's CDN (`cdn.shopify.com`). The extension migrates them through a durable pipeline so Webflow sites serve assets from Cloudflare R2 with full cache control.

| Stage | System | What Happens |
|-------|--------|--------------|
| **Ingest** | Shopify Admin API (paired) or CSV/Xano direct (standalone) | `products/images` fetched via GraphQL bulk query, or uploaded to Xano manually |
| **Normalize** | Xano (table 166) | Rows created per image: `product_id`, `src` (Shopify CDN), `alt`, `position`, `locale_alts` |
| **Mirror** | `asset-mirror-cdn` Worker | Downloads from Shopify CDN → writes to R2 bucket `prod-catalog` at `assets/{region}/{sku}/{hash}.{ext}` |
| **Transform** | Cloudflare Images (optional) | On-the-fly resize, format conversion (WebP/AVIF), quality tuning via URL params |
| **Deliver** | `asset-mirror-cdn` Worker | Serves from R2 with `Cache-Control: public, max-age=31536000, immutable`, CORS, range requests |
| **Bind** | Webflow CMS API | `src_r2` URL written to Webflow CMS collection item image field |

### 1.2 Supported Asset Types

| Type | Extensions | Storage Path | Max Size |
|------|-----------|--------------|----------|
| Product photos | `.jpg`, `.png`, `.webp`, `.avif` | `assets/{region}/{sku}/photo/{hash}.{ext}` | 20 MB |
| Lifestyle/editorial | `.jpg`, `.png` | `assets/{region}/{sku}/lifestyle/{hash}.{ext}` | 20 MB |
| Video | `.mp4`, `.webm` | `assets/{region}/{sku}/video/{hash}.{ext}` | 250 MB |
| 3D models | `.glb`, `.usdz`, `.stl`, `.obj`, `.3mf` | `assets/{region}/{sku}/3d/{hash}.{ext}` | 100 MB |
| SVG icons | `.svg` | `assets/{region}/{sku}/icon/{hash}.svg` | 1 MB |

### 1.3 Migration Controls (Extension UI)

| Control | Behavior |
|---------|----------|
| **Migrate All** | Queues every product image that has no `src_r2` value |
| **Migrate Selected** | Migrates only checked products from the collection browser |
| **Re-mirror** | Forces re-download even if `src_r2` already exists (hash comparison) |
| **Progress** | Live progress bar with count and current SKU |
| **Retry failed** | Re-queues items with `migration_status = "failed"` |

### 1.4 Failure Handling

- Transient HTTP errors (429, 502, 503): exponential backoff, max 3 retries
- Permanent failures (404 source gone): mark `migration_status = "source_missing"`, skip
- R2 write failures: retry once, then mark `migration_status = "r2_error"` with error detail
- All failures logged to Xano `sync_jobs` table (177) with `job_type = "asset_mirror"`

---

## 2. NLP/LLM-Optimized ALT Text and Tagging

### 2.1 ALT Text Generation

When a product image has an empty or generic `alt` field, the extension offers AI-generated alternatives.

| Feature | Specification |
|---------|--------------|
| **Trigger** | "Generate ALT" button per image, or "Generate All Missing" bulk action |
| **Model** | Cloudflare Workers AI — `@cf/meta/llama-3.2-11b-vision-instruct` (multimodal) |
| **Prompt template** | `Describe this product image for a blind user in under 120 characters. Product: {title}. Category: {product_type}. Do not start with "image of" or "photo of".` |
| **Locale support** | Generate per locale using `translate-proxy` worker for non-English locales |
| **Storage** | `alt` (primary locale) in table 166, `locale_alts` JSON map for translations |
| **Review flow** | Generated ALT shown inline with Accept / Edit / Reject actions |
| **Fallback** | If AI unavailable, suggest `{title} - {option1}` as template |

### 2.2 Auto-Tagging

| Feature | Specification |
|---------|--------------|
| **Trigger** | "Auto-tag" button per product or bulk "Tag All Untagged" |
| **Model** | Cloudflare Workers AI — `@cf/meta/llama-3.2-11b-vision-instruct` |
| **Prompt** | `List 3-5 product tags for this image. Product: {title}. Category: {product_type}. Return comma-separated lowercase tags.` |
| **Merge strategy** | Union with existing tags, deduplicate, normalize casing |
| **Tag taxonomy** | Validated against Shopify product_type and Google Product Categories |
| **Storage** | Merged into `tags` field on Xano table 164 |

### 2.3 Media Type Wrappers

Each media type gets an appropriate HTML wrapper when inserted into Webflow:

| Media Type | Wrapper Element | Key Attributes |
|------------|----------------|----------------|
| **Image** | `<img>` | `src`, `alt`, `width`, `height`, `loading="lazy"`, `decoding="async"` |
| **Video** | `<video>` | `src`, `poster`, `controls`, `preload="metadata"`, `playsinline` |
| **3D Model** | `<model-viewer>` | `src` (.glb), `ios-src` (.usdz), `alt`, `camera-controls`, `auto-rotate`, `ar` |
| **SVG** | `<img>` or inline `<svg>` | `role="img"`, `aria-label`, sanitized via DOMPurify |

---

## 3. Accessibility Attributes and Roles

### 3.1 Scope

Every element the extension inserts or modifies must comply with WCAG 2.2 AA. The extension enforces this at the template level — designers cannot accidentally strip required attributes.

### 3.2 UI Component Roles

| Component | Element | Required ARIA | Notes |
|-----------|---------|---------------|-------|
| Product card | `<article>` | `role="article"`, `aria-label="{title}"` | Card container |
| Image | `<img>` | `alt` (non-empty) | Falls back to `{title} - {option}` if ALT missing |
| Price | `<span>` | `aria-label="Price: {amount} {currency}"` | Screen reader gets formatted price |
| Sale price | `<span>` | `aria-label="Sale price: {sale}. Was: {original}"` | Both prices announced |
| Badge | `<span>` | `role="status"`, `aria-label="{badge text}"` | Sale / New / Bestseller |
| Add to Cart button | `<button>` | `aria-label="Add {title} to cart"`, `aria-disabled` when unavailable | Disabled state announced |
| Quick View trigger | `<button>` | `aria-haspopup="dialog"`, `aria-label="Quick view {title}"` | Announces modal intent |
| Quick View modal | `<div>` | `role="dialog"`, `aria-modal="true"`, `aria-label="{title}"` | Focus trapped inside |
| Variant selector | `<fieldset>` | `role="radiogroup"`, `aria-label="{option name}"` | Groups variant options |
| Variant option | `<button>` | `role="radio"`, `aria-checked`, `aria-label="{value}"` | Checked state tracked |
| Thumbnail slider | `<div>` | `role="tablist"`, `aria-label="Product images"` | Tab semantics for image strip |
| Thumbnail | `<button>` | `role="tab"`, `aria-selected`, `aria-controls="gallery-panel"` | Keyboard navigable |
| Gallery panel | `<div>` | `role="tabpanel"`, `id="gallery-panel"`, `aria-label="Selected image"` | Linked to active tab |
| Cart counter | `<span>` | `aria-live="polite"`, `aria-label="{n} items in cart"` | Announces count changes |
| Form error | `<span>` | `role="alert"`, `aria-live="assertive"` | Validation errors announced immediately |
| Loading spinner | `<div>` | `role="status"`, `aria-label="Loading"` | Non-visual loading indicator |

### 3.3 Keyboard Navigation

| Context | Keys | Behavior |
|---------|------|----------|
| Thumbnail slider | `ArrowLeft` / `ArrowRight` | Move between thumbnails |
| Variant selector | `ArrowLeft` / `ArrowRight` | Cycle through options |
| Quick View modal | `Escape` | Close modal, return focus to trigger |
| Quick View modal | `Tab` / `Shift+Tab` | Cycle within modal (focus trap) |
| Add to Cart | `Enter` / `Space` | Submit, announce confirmation |

### 3.4 Validation

The extension runs an accessibility audit before inserting any component:

| Check | Rule | Action on Fail |
|-------|------|----------------|
| Images without ALT | Every `<img>` must have non-empty `alt` | Block insert, prompt for ALT |
| Buttons without label | Every `<button>` needs visible text or `aria-label` | Auto-add `aria-label` from context |
| Color contrast | Text on backgrounds must meet 4.5:1 ratio | Warn in extension panel |
| Focus order | Tab order must be logical | Auto-set `tabindex` on inserted elements |

---

## 4. Form Validation and REGEX

### 4.1 Scope

All forms in the extension UI (config panel, product editor, sync settings) and all embed forms (contact, newsletter, custom) synced to Xano/Cloudflare metadata.

### 4.2 Validation Rules

| Field | Type | Regex / Rule | Error Message |
|-------|------|-------------|---------------|
| Xano API URL | URL | `^https:\/\/[\w-]+\.xano\.io\/api:[\w-]+` | "Must be a valid Xano API URL (https://...xano.io/api:...)" |
| Store domain | Domain | `^[\w-]+\.myshopify\.com$` | "Must be a .myshopify.com domain" |
| Shopify access token | Token | `^shp(at\|ua\|rt)_[a-f0-9]{32,}$` | "Invalid Shopify token format" |
| Webflow site ID | ID | `^[a-f0-9]{24}$` | "Must be a 24-character hex string" |
| Webflow collection ID | ID | `^[a-f0-9]{24}$` | "Must be a 24-character hex string" |
| Product handle | Slug | `^[a-z0-9]+(?:-[a-z0-9]+)*$` | "Lowercase letters, numbers, and hyphens only" |
| SKU | Alphanumeric | `^[A-Za-z0-9\-_.]{1,64}$` | "Letters, numbers, hyphens, dots, underscores (max 64)" |
| Price | Decimal | `^\d{1,7}(\.\d{1,2})?$` | "Valid price: up to 7 digits, 2 decimal places" |
| Email (embed forms) | Email | `^[^\s@]+@[^\s@]+\.[^\s@]{2,}$` | "Enter a valid email address" |
| Phone (embed forms) | Phone | `^\+?[\d\s\-().]{7,20}$` | "Enter a valid phone number" |
| Postal code | Varies | Region-specific regex map | "Enter a valid postal code for {country}" |
| Tag | Lowercase | `^[a-z0-9][a-z0-9\-_ ]{0,254}$` | "Tags: lowercase, max 255 chars" |
| Metafield namespace | Namespace | `^[a-z_][a-z0-9_]{0,19}$` | "Lowercase, underscores, max 20 chars" |
| Metafield key | Key | `^[a-z_][a-z0-9_]{0,29}$` | "Lowercase, underscores, max 30 chars" |

### 4.3 Validation Behavior

| Behavior | Specification |
|----------|--------------|
| **When** | On blur (single field) + on submit (all fields) |
| **Display** | Inline error below field, red border, `role="alert"` |
| **Sanitization** | Strip leading/trailing whitespace, normalize Unicode, escape HTML |
| **XSS prevention** | All user input passed through `escapeHtml()` before DOM insertion |
| **Submit block** | Form submit disabled until all fields pass validation |
| **Server-side echo** | Xano/Cloudflare endpoints re-validate; client errors are defense-in-depth |

### 4.4 Metadata Sync

Form submissions from Webflow embeds flow to Xano via the `webflow-sync` worker:

```
Webflow embed form → POST /form/{formId} → webflow-sync worker → Xano table → optional Shopify metafield sync
```

| Metadata Target | Table | Fields Synced |
|-----------------|-------|---------------|
| Product metafields | 164 | Custom fields mapped via merchant config (table 176) |
| Merchant config | 176 | `shop`, `webflow_site_id`, `webflow_collection_id`, tokens |
| Sync state | 179 | `shopify_gid`, `webflow_item_id`, `last_synced_at`, `status` |

---

## 5. Variant Image Thumbnail Preview Sliders

### 5.1 Gallery Architecture

Each PDP (Product Detail Page) embed includes a thumbnail strip + main image panel:

```
┌──────────────────────────────────┐
│          Main Image              │
│    (selected variant image)      │
│                                  │
│         800 x 800                │
│                                  │
├──┬──┬──┬──┬──┬──┬──┬────────────┤
│  │  │  │  │  │  │  │ ◀ strip ▶  │
└──┴──┴──┴──┴──┴──┴──┴────────────┘
 thumbnails — 80x80, scroll on overflow
```

### 5.2 Thumbnail Behavior

| Feature | Specification |
|---------|--------------|
| **Source** | `images[]` array from `/product?handle={handle}` endpoint |
| **Variant-linked** | When a variant is selected, the thumbnail strip scrolls to that variant's image and highlights it |
| **Lazy loading** | Thumbnails use `loading="lazy"`, main image uses `loading="eager"` |
| **Formats** | Prefer `src_r2` (Cloudflare R2) over `src` (Shopify CDN) |
| **Fallback** | If no variant image, show product's first image |
| **Zoom** | Click main image to open lightbox (pinch-zoom on mobile) |
| **Touch** | Swipe left/right on thumbnail strip (mobile), `-webkit-overflow-scrolling: touch` |
| **Keyboard** | Arrow keys navigate thumbnails, Enter selects |
| **Count badge** | If >7 images, show "+N more" on last visible thumbnail |

### 5.3 Variant → Image Mapping

```
variant.image_url → find matching image in images[] by URL
                  → if no match, use images[0]
                  → highlight corresponding thumbnail
                  → scroll strip to show active thumbnail
```

### 5.4 CSS Custom Properties

All slider dimensions are themeable via PIM CSS variables (loaded from `/embed/header`):

| Variable | Default | Purpose |
|----------|---------|---------|
| `--pim-gallery-main-size` | `800px` | Main image max dimension |
| `--pim-gallery-thumb-size` | `80px` | Thumbnail dimension |
| `--pim-gallery-thumb-gap` | `8px` | Gap between thumbnails |
| `--pim-gallery-thumb-border` | `2px solid var(--pim-color-border)` | Inactive thumbnail border |
| `--pim-gallery-thumb-active-border` | `2px solid var(--pim-color-primary)` | Active thumbnail border |
| `--pim-gallery-radius` | `var(--pim-radius-md)` | Border radius on images |

---

## 6. Commerce Widgets

### 6.1 Add to Cart

| Feature | Specification |
|---------|--------------|
| **Checkout method** | **Paired:** Shopify Storefront API `checkoutCreate` with authenticated session. **Standalone:** Shopify Buy Button JS SDK with manually configured Storefront Access Token. |
| **Variant selection** | Must select a valid variant before enabling button |
| **Quantity** | Numeric input with +/- steppers, min=1, max=inventory |
| **Inventory check** | Real-time via Storefront API `quantityAvailable` |
| **Disabled state** | `aria-disabled="true"` + visual grey-out when variant unavailable |
| **Success feedback** | Button text changes to "Added" for 2s, cart counter increments |
| **Error handling** | Toast notification for API failures, button re-enabled |
| **Multi-currency** | Price displayed in visitor's market currency (from FX cache) |

### 6.2 Quick View Modal

| Feature | Specification |
|---------|--------------|
| **Trigger** | "Quick View" button on product card in collection grid |
| **Content** | Product title, image gallery (mini slider), variant selector, price, Add to Cart, description excerpt |
| **Data source** | `/product?handle={handle}` endpoint on `webflow-sync` worker |
| **Open animation** | Fade-in overlay + scale-up modal (CSS transitions, respects `prefers-reduced-motion`) |
| **Close** | X button, click overlay, Escape key |
| **Focus trap** | Tab cycles within modal; focus returns to trigger on close |
| **Scroll lock** | `document.body.style.overflow = "hidden"` while open |
| **Mobile** | Full-screen slide-up sheet instead of centered modal |
| **Lazy load** | Modal content fetched on trigger click, not on page load |

### 6.3 Cart Manager

| Feature | Specification |
|---------|--------------|
| **Storage** | `localStorage` key `pim-cart-{siteId}` — array of `{ variantId, quantity, handle, title, price, image }` |
| **Sync** | On page load, validate cart items against Storefront API (remove deleted/unavailable). Standalone: validate against Xano product status only. |
| **UI** | Slide-out drawer from right edge |
| **Line items** | Image thumbnail, title, variant label, quantity stepper, line total, remove button |
| **Subtotal** | Calculated client-side, formatted per locale |
| **Checkout** | "Checkout" button creates Storefront API checkout and redirects to Shopify checkout |
| **Empty state** | Illustration + "Your cart is empty" + "Continue Shopping" link |
| **Persistence** | Cart survives page navigation and browser refresh |
| **Cross-page** | Cart counter in header updated via `BroadcastChannel` or `storage` event |
| **Max items** | 50 line items, quantity max 99 per item |

### 6.4 Embed Code Generation

The extension generates copy-paste embed snippets for each widget:

| Widget | Embed Type | Placement |
|--------|-----------|-----------|
| Header CSS Variables | `<style>` + `<script>` | Site Settings → Head Code |
| Collection Grid | `<script>` | Site Settings → Footer Code |
| PDP Variants | `<div>` + `<script>` | Code Embed on product template |
| Quick View | Included in Collection Grid embed | Automatic |
| Cart Drawer | Included in Header CSS embed | Automatic |
| Add to Cart | Included in PDP Variants embed | Automatic |

---

## 7. Tag Reconciliation and Dedupe

### 7.1 Problem

Product tags accumulate duplicates and inconsistencies across three systems (Shopify, Xano, Webflow): mixed casing, trailing spaces, singular/plural variants, deprecated terms.

### 7.2 Reconciliation Pipeline

```
Shopify tags (source of truth)
    ↓ full_sync
Xano table 164 `tags` field (comma-separated string)
    ↓ webflow_sync
Webflow CMS `tags` field (multi-reference or plain text)
```

### 7.3 Cleanup Rules

| Rule | Example | Action |
|------|---------|--------|
| **Normalize case** | `Leather`, `leather`, `LEATHER` → `leather` | Lowercase all |
| **Trim whitespace** | `" cotton "` → `"cotton"` | Strip leading/trailing spaces |
| **Deduplicate** | `["red", "red", "Red"]` → `["red"]` | Case-insensitive unique |
| **Singularize** | `"accessories"` + `"accessory"` → `"accessories"` (keep plural) | Configurable: keep longer form |
| **Remove empties** | `["", " ", "red"]` → `["red"]` | Filter blank entries |
| **Max length** | Tag > 255 chars → truncate | Shopify limit enforcement |
| **Max count** | > 250 tags per product → warn | Shopify limit enforcement |
| **Blocklist** | Merchant-defined terms to auto-remove | Checked on every sync |

### 7.4 Reconciliation UI (Extension Panel)

| Feature | Specification |
|---------|--------------|
| **Tag cloud** | Visual display of all tags with frequency count |
| **Duplicates view** | Side-by-side list of duplicate clusters with merge action |
| **Blocklist editor** | Add/remove terms that should never appear as tags |
| **Bulk rename** | Rename a tag across all products in one action |
| **Preview** | Show before/after diff before applying changes |
| **Apply** | Writes cleaned tags back to Xano → syncs to Shopify and Webflow |

### 7.5 Auto-Reconciliation (Scheduled)

- Runs as part of each `full_sync` job
- Applies normalize + trim + dedup + remove empties automatically
- Singularize and blocklist only applied if merchant has enabled them in settings
- Results logged to `sync_jobs` table with `job_type = "tag_reconciliation"`

---

## 8. Three-Way Data Mirror (Paired Mode) / Two-Way Mirror (Standalone)

### 8.1 Architecture

> In **standalone mode**, the Shopify layer is absent. The mirror is Xano ↔ Webflow only, triggered by manual sync and Webflow webhooks. Shopify publish and reverse sync features are disabled.

```
                  ┌───────────┐
        full_sync │           │ publish_sync
    ────────────▶ │   Xano    │ ────────────▶
                  │ (tables   │
    ◀──────────── │ 164,166)  │ ◀────────────
      webhook     │           │  webflow_sync
                  └─────┬─────┘
                        │
              webflow   │  webflow
              _sync     │  webhook
                        │
                  ┌─────▼─────┐
                  │  Webflow  │
                  │   CMS     │
                  └───────────┘
```

| Direction | Trigger | Worker | Behavior |
|-----------|---------|--------|----------|
| **Shopify → Xano** | `products/create`, `products/update` webhook | Shopify app `_worker.js` | Upsert product/variant/image rows in tables 164–166. *Owned by Shopify app.* |
| **Xano → Webflow** | Manual sync or scheduled cron | `webflow-sync` | Push Xano products to Webflow CMS via REST API. *Owned by extension.* |
| **Webflow → Xano** | Webflow `collection_item_changed` webhook | `webflow-sync` | Update Xano from Webflow changes. *Owned by extension.* |
| **Webflow → Xano → Shopify** | Webflow webhook + reverse sync | `webflow-sync` | Push Webflow changes to Xano; Shopify app picks up changes on next sync. *Extension writes to Xano only; Shopify app owns the Admin API mutation.* |
| **Xano → Shopify** | Manual publish or scheduled | Shopify app `publish-sync` | Register translations, set metafields, update core fields. *Owned by Shopify app — extension does not call Admin API directly.* |

### 8.2 Echo Detection

Bidirectional sync creates echo loops (Shopify updates Webflow, Webflow webhook fires, tries to update Shopify). The system prevents this with two mechanisms:

| Mechanism | Storage | TTL | Check |
|-----------|---------|-----|-------|
| **KV marker** | `WEBFLOW_SYNC_STATE` KV | 120s | Before processing Webflow webhook, check `shopify_sync:{itemId}` exists |
| **Xano state** | Table 179 `last_sync_direction` | 120s window | `isEchoSync()` compares direction + timestamp |

### 8.3 Conflict Detection

When both sides change the same field within 120s (after echo detection), the system flags a conflict:

| Scenario | Detection | Resolution |
|----------|-----------|------------|
| Same field, same value | No conflict | Skip (idempotent) |
| Same field, different values | Conflict flagged | Stored in Xano table 178 (`webflow_conflicts`) |
| Different fields | No conflict | Both changes applied |

Conflict resolution is manual: the Shopify app's Webflow integration page shows conflicts with "Keep Shopify" / "Keep Webflow" actions per field.

### 8.4 Cron / Scheduled Sync

| Job | Schedule | Worker | What It Does |
|-----|----------|--------|-------------|
| **FX rate refresh** | Every 6 hours | `fx-cache` | Fetch ECB + Bank of Canada rates, recalculate market prices |
| **Omnibus audit** | Daily at 02:00 UTC | `omnibus-audit` | Check 30-day price windows, flag violations |
| **Full catalog sync** | Manual or daily | Pages `_worker.js` | Re-import all Shopify products to Xano |
| **Webflow push** | Manual or after full sync | `webflow-sync` | Push all Xano products to Webflow CMS |
| **Tag reconciliation** | On every full sync | Pages `_worker.js` | Normalize, dedup, apply blocklist |
| **Asset mirror** | On every full sync | `asset-mirror-cdn` | Mirror new/changed images to R2 |

### 8.5 Logging

Every sync operation writes to Xano table 177 (`sync_jobs`):

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Auto-increment |
| `shop` | string | Merchant domain |
| `job_type` | enum | `full_sync`, `webflow_push`, `webflow_pull`, `asset_mirror`, `tag_reconciliation`, `publish`, `fx_refresh`, `omnibus_audit` |
| `direction` | enum | `shopify_to_xano`, `xano_to_webflow`, `webflow_to_xano`, `xano_to_shopify` |
| `status` | enum | `pending`, `running`, `completed`, `failed` |
| `product_count` | int | Items processed |
| `error_count` | int | Items failed |
| `error_message` | text | Last error detail (truncated to 2000 chars) |
| `started_at` | datetime | Job start |
| `completed_at` | datetime | Job end |
| `created_at` | datetime | Record creation |

Logs are queryable from the Shopify app's sync management page and the extension's status panel.

---

## 9. Non-Destructive Publish (Upsert to Live Shopify)

> The Xano → Shopify publish pipeline (translations, metafields, core fields) is handled by the **PIM Sync Shopify app** already live in the Shopify App Store. The Webflow extension does not duplicate those mutations. This section documents the contract the extension relies on when triggering a publish via the Shopify app's existing workers.

### 9.1 Principle

Publishing from PIM Sync to a live Shopify store must never break the storefront. The Shopify app's `publish-sync` worker owns all Admin API mutations — the extension triggers publishes by writing to Xano, which the Shopify app picks up.

### 9.2 Safety Guarantees

| Guarantee | Implementation |
|-----------|---------------|
| **No deletes** | PIM Sync never calls `productDelete`. Unpublish only removes from sales channels. |
| **Upsert only** | All mutations use `productUpdate` (not `productCreate` + `productDelete`). New products use `productCreate` only when no matching `handle` exists. |
| **Field-level granularity** | Only changed fields are included in the mutation. Unchanged fields are omitted, not sent as current values. |
| **Draft-first** | New products are created with `status: DRAFT`. Merchant must explicitly publish to Online Store. |
| **Metafield append** | `metafieldsSet` is additive — it creates or updates specific keys without touching other metafields on the product. |
| **Translation append** | `translationsRegister` sets specific locale keys without removing existing translations. |
| **Price safety** | Price changes require merchant confirmation in the Shopify app UI. The extension cannot silently change prices. |
| **Image append** | New images are appended to the product's media list. Existing images are only updated (alt text, position), never deleted. |
| **Variant preservation** | Existing variants are updated in place. New variants are appended. Variants are never deleted by sync. |

### 9.3 Pre-Publish Validation

Before any publish operation, the system runs these checks:

| Check | What | Fail Action |
|-------|------|-------------|
| **Handle collision** | Does a different product already have this handle? | Block publish, surface in UI |
| **Required fields** | Title, at least one variant, at least one image | Block publish, list missing fields |
| **Price sanity** | Price > 0, compare_at_price > price (if set) | Warn, allow override |
| **Inventory consistency** | Variant inventory levels sum correctly | Warn |
| **Image availability** | All `src_r2` URLs return 200 | Warn, publish with Shopify CDN fallback |
| **Tag limits** | ≤ 250 tags, each ≤ 255 chars | Auto-truncate, warn |
| **Metafield types** | Values match declared metafield type | Block publish for type mismatches |
| **Scope check** | App has required API scopes | Block publish, show missing scopes |

### 9.4 Publish Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Preview** | Dry run — shows what would change without writing | Pre-publish review |
| **Selective** | Publish only selected products | Targeted updates |
| **Incremental** | Publish only products changed since last sync | Regular cadence |
| **Full** | Publish entire catalog | Initial setup or recovery |

### 9.5 Rollback

| Scenario | Rollback Method |
|----------|----------------|
| Bad translation | Re-run `translationsRegister` with original values from Xano |
| Wrong price | Merchant corrects in Shopify admin; next sync pulls corrected price back to Xano |
| Broken metafield | `metafieldsSet` with corrected value; or `metafieldDelete` if field should not exist |
| Accidental unpublish | Re-publish to sales channel via `publishablePublish` mutation |

### 9.6 Publish Sequence (Extension Role)

The extension's responsibility ends at Xano. The Shopify app's `publish-sync` worker handles all Admin API mutations.

```
Extension side:
1. Validate: Run pre-publish checks (§9.3) against Xano data
2. Diff: Compare Xano state vs last known Shopify state (from sync_jobs)
3. Write: Update Xano tables with cleaned/validated data
4. Trigger: Create a sync_jobs record with job_type="publish", status="pending"

Shopify app side (already live):
5. Pick up: publish-sync worker reads pending publish jobs from Xano
6. Batch: Group Shopify mutations (rate limit: 2 req/s with cost budgeting)
7. Execute: Send mutations (translationsRegister, metafieldsSet, productUpdate)
8. Echo-mark: Write KV markers so webhooks don't loop back
9. Complete: Update sync_jobs status to "completed" or "failed"
```

---

## 10. Extension UI Layout

### 10.1 Tab Structure

| Tab | Contents |
|-----|----------|
| **Products** | Collection browser, product list, filter by collection, select/insert actions |
| **Assets** | Migration controls, progress, ALT text generator, image preview grid |
| **Commerce** | Variant slider preview, Add to Cart config, Quick View config, Cart Manager settings |
| **Tags** | Tag cloud, duplicate finder, blocklist, bulk rename, reconciliation actions |
| **Sync** | 3-way mirror status, last sync timestamps, job history, manual trigger buttons |
| **Embeds** | Copy-paste code generators for Header, Grid, PDP, Quick View, Cart |
| **Config** | Mode selector (Standalone / Paired), Xano URL, store domain, access token (auto-filled in paired mode), Storefront Access Token (standalone), Webflow site/collection IDs, validation status |

### 10.2 Status Bar

Persistent bar at bottom of extension panel:

```
[Site: {siteName}] [Sync: {lastSyncTime}] [Products: {count}] [Status: {connected|error}]
```

---

## 11. Compliance

### 11.1 Data Classification

| Data Category | Examples | Classification | Stored Where |
|---------------|----------|---------------|-------------|
| Product catalog | Titles, descriptions, handles, SKUs, prices | Business data — non-sensitive | Xano tables 164–166, Webflow CMS, R2 |
| Product media | Images, videos, 3D models, ALT text | Business data — non-sensitive | Cloudflare R2, Xano table 166 |
| Merchant credentials | Shopify access tokens, Xano API keys, Webflow OAuth tokens | Secret — encrypted at rest | Xano table 175 (sessions), Cloudflare KV |
| Merchant config | Shop domain, target markets, pricing rules, collection IDs | Business config — low sensitivity | Xano table 176 |
| Sync metadata | Job history, timestamps, error messages, conflict records | Operational data | Xano tables 177–179, Cloudflare KV |
| Visitor cart | Variant IDs, quantities, prices | Ephemeral — no PII | Browser `localStorage` only |
| AI-generated content | ALT text, auto-tags | Derived data — non-sensitive | Xano tables 164, 166 |

### 11.2 Personal Data Handling

| Requirement | Implementation |
|-------------|---------------|
| **No customer PII** | The extension does not access, process, or store any customer personal data. Shopify scopes are product-only (`read_products`, `write_products`). Customer scopes (`read_customers`, `write_customers`) belong exclusively to the CRM Sync app. |
| **No visitor tracking** | Embeds do not set cookies, fingerprint browsers, or collect analytics. No third-party trackers injected. |
| **No IP logging** | Workers log request metadata (path, status, timing) but strip client IP before writing to Xano. |
| **Cart data** | Stored in `localStorage` on the visitor's device. Never transmitted to Xano or any backend except during Shopify checkout redirect (Storefront API only). |
| **AI model input** | Product text and images sent to Cloudflare Workers AI for ALT text and tagging. No PII is present in product data. Cloudflare does not retain inference inputs beyond the request lifecycle (Workers AI data usage policy). |

### 11.3 GDPR Compliance

| Article | Requirement | How the Extension Complies |
|---------|-------------|---------------------------|
| Art. 5(1)(c) | Data minimization | Only product catalog data stored. No customer data. No behavioral tracking. |
| Art. 5(1)(e) | Storage limitation | Sync job logs retained 90 days, then purged. Conflict records purged on resolution. Asset migration logs purged after 30 days. |
| Art. 17 | Right to erasure | `shop/redact` webhook (handled by Shopify app) triggers full purge of merchant data from Xano tables 164–166, 175–179 within 48 hours. R2 assets deleted in same sweep. |
| Art. 25 | Data protection by design | Tokens encrypted at rest in Cloudflare KV. API group isolation prevents cross-app data access. Extension cannot read CRM tables (180–183). |
| Art. 28 | Processor obligations | Xano (processor) operates under Anthropic-standard DPA. Cloudflare operates under Cloudflare DPA. No sub-processor stores customer PII. |
| Art. 30 | Records of processing | Sync job table (177) serves as processing activity log with timestamps, data categories, and purposes. |
| Art. 32 | Security of processing | TLS in transit, encrypted at rest, API key rotation, HMAC webhook validation, CORS restrictions. |
| Art. 44 | International transfers | Xano data center: US. Cloudflare R2: multi-region with data locality controls. No EU customer PII transferred — product catalog data only. |

### 11.4 CCPA / CPRA Compliance

| Requirement | Implementation |
|-------------|---------------|
| No sale of personal information | Extension does not collect or sell consumer personal information |
| Service provider status | Xano and Cloudflare act as service providers, not third parties |
| Data deletion | Same mechanism as GDPR Art. 17 — `shop/redact` purges all merchant data |

### 11.5 EU Omnibus Directive (2019/2161)

Omnibus compliance is enforced by the Shopify app's `omnibus-audit` worker. The extension surfaces compliance data in two ways:

| Feature | Location | Behavior |
|---------|----------|----------|
| **Price history badge** | PDP embed | If `omnibus_price` metafield exists, display "Lowest price in last 30 days: {price}" below sale price |
| **Audit flag** | Extension Sync tab | Products flagged by `omnibus-audit` shown with warning icon and "Review pricing" link to Shopify app |
| **Compare-at validation** | Pre-publish check (§9.3) | Block publish if `compare_at_price` set but no valid 30-day price history exists |

### 11.6 PCI DSS

| Requirement | Implementation |
|-------------|---------------|
| **No card data** | The extension never handles, transmits, or stores payment card data. All checkout flows redirect to Shopify's PCI-compliant checkout. |
| **Storefront API only** | Cart and checkout operations use Shopify Storefront API, which returns checkout URLs — no card fields rendered by extension code. |
| **No custom payment forms** | Extension embeds use Shopify Buy Button SDK or Storefront API `checkoutCreate`. No `<input type="card">` or equivalent. |

### 11.7 Accessibility Compliance (WCAG 2.2 AA)

Covered in detail in §3. Summary of obligations:

| Standard | Level | Status |
|----------|-------|--------|
| WCAG 2.2 | AA | All inserted components comply. Audit-before-insert enforced. |
| Section 508 | — | Satisfied by WCAG 2.2 AA conformance |
| EN 301 549 | — | EU public sector accessibility — satisfied by WCAG 2.2 AA |
| ADA Title III | — | US web accessibility — satisfied by WCAG 2.2 AA |

### 11.8 Webflow Marketplace Requirements

| Requirement | Implementation |
|-------------|---------------|
| Designer API v2 | Extension uses `apiVersion: "2"` in `webflow.json` |
| No external auth wall | Extension functions immediately after install — config is optional enhancement |
| No data collection on install | No forms or consent prompts on first load |
| Graceful degradation | If Xano is unreachable, extension shows connection status and cached product count |
| Uninstall cleanup | Extension stores data in `localStorage` only — browser-scoped, auto-cleaned |

### 11.9 Content Licensing and AI Disclosure

| Concern | Policy |
|---------|--------|
| AI-generated ALT text | Clearly labeled as "AI-generated" in extension UI with Accept/Edit/Reject flow. Merchant is responsible for reviewing before publish. |
| AI-generated tags | Labeled "Suggested by AI" — merged only on explicit merchant action |
| Image rights | Extension mirrors images the merchant already owns in Shopify. No scraping, no stock photo injection. |
| Model training | Product images sent to Workers AI are not used for model training (Cloudflare Workers AI terms of service). |

---

## 12. Security

### 12.1 Authentication and Authorization

| Layer | Mechanism | Details |
|-------|-----------|---------|
| **Extension → Xano** | Bearer token | `XANO_API_KEY` scoped to PIM API group (`api:Ksv9J-nS`). Cannot access CRM tables 180–183. |
| **Extension → Webflow** | OAuth 2.0 | Webflow app OAuth with site-scoped access. Token stored in Xano table 176. |
| **Shopify app → Workers** | HMAC-SHA256 | All Shopify webhooks validated against `SHOPIFY_API_SECRET` |
| **Webflow → Workers** | HMAC-SHA256 | Webflow webhooks validated against `WEBFLOW_WEBHOOK_SECRET` |
| **Embed → Workers** | Public (read-only) | `/collection`, `/product`, `/search`, `/embed/*` are public GET endpoints. No mutations exposed publicly. |
| **Config endpoints** | Bearer token | `POST /config` requires valid Xano API key in request chain |
| **Echo-mark endpoint** | Shared secret | `POST /echo-mark` validated against `ECHO_SECRET` header |

### 12.2 API Group Isolation

| Boundary | Enforcement |
|----------|------------|
| PIM group `api:Ksv9J-nS` → CRM tables (180–183) | Blocked by Xano API group scoping. PIM API key has zero access to CRM tables. |
| CRM group `api:1Zsx4CNw` → PIM tables (164–179) | Blocked. CRM API key has zero access to PIM tables. |
| Cross-worker | Each worker has its own `XANO_API_KEY` env var. Key rotation on one app does not affect the other. |

### 12.3 Transport Security

| Requirement | Implementation |
|-------------|---------------|
| TLS everywhere | All API calls (Xano, Webflow, Shopify, Cloudflare) over HTTPS. No HTTP fallback. |
| Certificate pinning | Not applicable — Cloudflare Workers runtime handles TLS termination. |
| HSTS | Workers set `Strict-Transport-Security: max-age=31536000; includeSubDomains` on embed responses. |
| Minimum TLS version | TLS 1.2 (Cloudflare default). TLS 1.0/1.1 rejected at edge. |

### 12.4 Input Validation and Injection Prevention

| Attack Vector | Mitigation |
|---------------|-----------|
| **XSS** | All dynamic values passed through `escapeHtml()` before DOM insertion. Embed code uses `textContent` for user data, not `innerHTML`. |
| **SQL injection** | N/A — Xano uses parameterized queries internally. Extension never constructs raw SQL. |
| **Command injection** | N/A — Workers runtime has no shell access. |
| **Path traversal** | `asset-mirror-cdn` rejects keys not starting with `assets/`. R2 key constructed from sanitized `{region}/{sku}/{hash}.{ext}` — no `..` sequences allowed. |
| **Prototype pollution** | JSON payloads parsed with `JSON.parse()` (safe). No `Object.assign` from untrusted input. |
| **Open redirect** | No user-controlled redirects in worker code. Checkout URLs are Shopify-origin only. |
| **SSRF** | Workers do not accept user-provided URLs for server-side fetch. Xano base URL is an env var, not user input. |

### 12.5 SVG and Media Sanitization

| File Type | Sanitization |
|-----------|-------------|
| **SVG** | Before inline injection: strip `<script>`, `<iframe>`, `<object>`, `<embed>`, `<foreignObject>`, all `on*` event handlers, `javascript:` URIs, `data:` URIs (except `data:image/`). Validated against allowlist of safe elements and attributes. |
| **3D models (GLB/USDZ)** | Served as binary download via R2. Never parsed or executed server-side. `model-viewer` handles rendering in a sandboxed context. |
| **Images (JPEG/PNG/WebP/AVIF)** | Served with correct `Content-Type`. No server-side processing beyond R2 storage. Cloudflare Images handles transforms at edge. |
| **Video (MP4/WebM)** | Served with `Content-Type: video/mp4` or `video/webm`. No transcoding. Range requests supported for streaming. |

### 12.6 CORS Policy

| Endpoint Category | `Access-Control-Allow-Origin` | Methods | Rationale |
|-------------------|-------------------------------|---------|-----------|
| Public data (`/collection`, `/product`, `/search`, `/embed/*`) | `*` | `GET`, `HEAD` | Read-only product data, intentionally public for embed consumption |
| Media (`/media/*`) | `*` | `GET`, `HEAD` | Public CDN assets |
| Config (`/config`) | `*` | `GET`, `POST` | Called from extension (Webflow Designer sandbox — origin varies). Auth via API key in request chain. |
| Webhooks (`POST /`) | N/A | `POST` | Server-to-server. HMAC-validated. |
| Echo endpoints | `*` | `GET`, `POST` | Internal worker-to-worker. Protected by `ECHO_SECRET`. |

### 12.7 Secrets Management

| Secret | Storage | Rotation |
|--------|---------|----------|
| `SHOPIFY_API_SECRET` | Cloudflare Pages env (encrypted) | On compromise — regenerate in Shopify Partners dashboard |
| `XANO_API_KEY` (PIM) | Cloudflare Workers env (encrypted) + Xano dashboard | Rotatable independently of CRM key |
| `WEBFLOW_WEBHOOK_SECRET` | Cloudflare Workers env (encrypted) | On compromise — regenerate in Webflow app dashboard |
| `ECHO_SECRET` | Cloudflare Workers env (encrypted) | Shared between `webflow-sync` and `publish-sync` workers |
| Shopify access tokens (`shpua_`) | Cloudflare KV (`SESSIONS`) + Xano table 175 | Auto-rotated every 60 min by Shopify app's token refresh |
| Webflow OAuth token | Xano table 176 | Rotated on re-authorization |

**Key rotation procedure:**
1. Generate new key in source system (Xano dashboard, Cloudflare dashboard, Shopify Partners)
2. Update Cloudflare Worker/Pages env var via `wrangler secret put`
3. Deploy workers with `wrangler deploy` (zero-downtime — old key valid until KV propagation completes)
4. Verify: hit `/config?shop={domain}` to confirm new key works
5. Revoke old key in source system

### 12.8 Rate Limiting and Abuse Prevention

| API | Limit | Strategy |
|-----|-------|----------|
| Shopify Admin GraphQL | 1000 cost points/sec | Cost budgeting per query. Exponential backoff on 429. Managed by Shopify app, not extension. |
| Shopify Storefront API | 100 req/sec per app | Client-side throttle in embed JS. Cart operations debounced 500ms. |
| Xano | 100 req/sec per API group | Batch table reads. KV cache for repeated lookups (300s TTL). |
| Webflow REST API | 60 req/min | Chunked sync with 1s delay between batches. Progress bar in extension UI. |
| Cloudflare Workers AI | Per-plan limits | Queue model requests. Fallback to template-based ALT text if quota exhausted. |
| `webflow-sync` public endpoints | No built-in rate limit | Cloudflare rate limiting rules applied at zone level. Configurable per merchant. |

### 12.9 Dependency Security

| Practice | Implementation |
|----------|---------------|
| Lockfile pinning | `package-lock.json` committed. Exact versions installed. |
| Audit | `npm audit` run before every deploy. Critical/high vulnerabilities block deployment. |
| Minimal dependencies | Extension has 2 runtime dependencies: TypeScript (build-only), ESLint (dev-only). No runtime npm packages in the bundle. |
| Supply chain | No post-install scripts. No CDN-loaded JS in embeds — all code is self-contained in the worker or extension bundle. |
| Webflow Designer API | Loaded from Webflow's own runtime (sandboxed iframe). Extension cannot access parent frame DOM. |

### 12.10 Incident Response

| Scenario | Detection | Response |
|----------|-----------|----------|
| **Leaked API key** | Xano audit log shows unauthorized access, or GitHub secret scanning alert | Rotate key immediately (§12.7). Review Xano audit log for unauthorized reads/writes. Notify affected merchant if data was accessed. |
| **Webhook spoofing** | HMAC validation fails — logged as 401/403 in worker logs | No data modified (rejected at validation). Monitor for volume — if sustained, rotate webhook secret. |
| **XSS in embed** | User report or automated scan | Patch `escapeHtml()` or template. Redeploy workers. Bust CDN cache for affected embed paths (`Cache-Control: no-store` temporarily). |
| **R2 data breach** | Cloudflare access logs show unauthorized reads | R2 bucket is public-read for `assets/` prefix only. No PII stored. Rotate R2 credentials if bucket-level access compromised. |
| **Xano outage** | Worker health checks return 500 | Extension shows "Connection error" status. Cached product data served from KV for up to 300s. Sync operations queued, not lost. |

### 12.11 Logging and Audit Trail

| What is Logged | Where | Retention | PII |
|----------------|-------|-----------|-----|
| Sync job execution (start, end, counts, errors) | Xano table 177 | 90 days | No |
| Webhook receipt (trigger type, item ID, timestamp) | Worker `console.log` → Cloudflare Logpush | 7 days | No |
| Config changes (shop, field mappings) | Xano table 176 `updated_at` | Indefinite (merchant config) | Shop domain only |
| Conflict records | Xano table 178 | Until resolved, then purged | No |
| Echo markers | Cloudflare KV | 120s TTL (auto-expired) | No |
| Failed auth attempts (HMAC mismatch) | Worker `console.error` → Cloudflare Logpush | 7 days | No |
| Asset migration results | Xano table 177 (`job_type = "asset_mirror"`) | 90 days | No |

**What is never logged:**
- Shopify access tokens (`shpua_`, `shprt_`)
- Xano API keys
- Webflow OAuth tokens
- Request bodies containing credentials
- Visitor IP addresses
- Customer personal data (not applicable — extension has no customer data)

---

## 13. Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| Webflow Designer API | v2 | Element manipulation, component registration, page navigation |
| Shopify Storefront API | 2026-04 | Cart operations, inventory checks, checkout creation (embed-side) |
| Shopify Admin API | 2026-04 | Product mutations, translations, metafields (owned by Shopify app workers, not extension) |
| Cloudflare Workers AI | latest | Vision model for ALT text and auto-tagging |
| Cloudflare R2 | — | Asset storage |
| Cloudflare KV | — | Sync state, echo markers, cache |
| Xano Meta API | — | Table CRUD for product data, config, sync state |
| `model-viewer` | 4.x | 3D model rendering in browser |
