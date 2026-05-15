# PIM Sync Webflow Extension — Explainer Video Script

**Duration:** ~4:30  
**Tone:** Professional, direct, technical-but-accessible  
**Audience:** Webflow designers and Shopify merchants evaluating PIM Sync  
**Resolution:** 1920x1080 (16:9)  
**Music:** Subtle ambient electronic (lower volume under narration)

---

## Scene 1 — Title (0:00–0:08)

**Visual:** Dark background. PIM Sync logo fades in center. Tagline types out below.

**Narration:**
"PIM Sync. One extension. Three systems in sync. Your products, everywhere."

---

## Scene 2 — The Problem (0:08–0:30)

**Visual:** Three platform icons (Shopify, Xano, Webflow) appear disconnected with red dashed lines. Data icons (images, prices, tags) scatter between them with conflict markers.

**Narration:**
"If you're running a Shopify store with a Webflow frontend, you know the pain. Product data lives in three places — Shopify for commerce, Xano for your content API, and Webflow for your CMS. Change a price in Shopify, and Webflow is already stale. Upload an image in Webflow, and Shopify doesn't know it exists. Tags drift. ALT text goes missing. And every sync is a manual export."

---

## Scene 3 — The Solution (0:30–0:50)

**Visual:** The three icons reconnect with solid blue lines forming a triangle. Data flows smoothly in animated arrows. The triangle pulses once and settles.

**Narration:**
"PIM Sync is a Webflow Designer Extension that keeps all three systems in lock-step. It works standalone with just Xano and Webflow, or paired with the PIM Sync Shopify app for the full three-way mirror. Install it, connect your Xano API, and your product catalog is live inside the Webflow Designer."

---

## Scene 4 — Asset Migration (0:50–1:15)

**Visual:** Pipeline diagram animates left to right: Shopify CDN icon to Xano table row to Cloudflare R2 bucket to Webflow CMS card. Progress bar fills underneath.

**Narration:**
"Asset migration runs through a durable pipeline. Product images flow from Shopify through Xano into Cloudflare R2 — your own CDN with immutable caching, range requests, and automatic format conversion. Every image gets a permanent, cache-friendly URL. Failed transfers retry automatically. And the extension shows you progress in real time — no guessing which images made it."

---

## Scene 5 — AI-Powered ALT Text and Tagging (1:15–1:45)

**Visual:** Product image appears. A shimmer effect scans across it. ALT text types out below: "Matte black wireless headphones with rotating ear cups on white background." Tag chips animate in: "headphones", "wireless", "matte-black".

**Narration:**
"Every image needs ALT text. Writing it by hand for hundreds of products isn't realistic. PIM Sync uses Cloudflare Workers AI to generate descriptive, accessible ALT text from your product images — optimized for screen readers, not stuffed with keywords. It suggests tags too, validated against your existing taxonomy. You review, accept or edit, and they sync everywhere. All AI-generated content is clearly labeled — you stay in control."

---

## Scene 6 — Accessibility (1:45–2:10)

**Visual:** Product card blueprint appears with ARIA labels annotated: `role="article"`, `aria-label`, `aria-haspopup="dialog"`. Keyboard focus ring animates through card elements. Checkmarks appear next to each ARIA attribute.

**Narration:**
"Every component PIM Sync inserts is WCAG 2.2 double-A compliant out of the box. Product cards carry proper ARIA roles. Variant selectors are keyboard-navigable. Quick View modals trap focus. Image thumbnails use tab semantics. And the extension audits accessibility before insertion — if an image has no ALT text, it blocks the insert and prompts you to fix it. Compliance isn't optional."

---

## Scene 7 — Variant Thumbnail Slider (2:10–2:30)

**Visual:** PDP layout appears — main image area on top, thumbnail strip below. Clicking a color swatch swaps the main image and highlights the corresponding thumbnail. The strip scrolls to reveal more thumbnails.

**Narration:**
"Product detail pages get a variant-linked image gallery. Select a color, and the thumbnail strip scrolls to that variant's image automatically. All dimensions are themeable through CSS custom properties — your design system, not ours. Touch-swipe on mobile, arrow keys on desktop, lazy-loaded thumbnails, and zoom on click."

---

## Scene 8 — Commerce Widgets (2:30–2:55)

**Visual:** Three panels animate in sequence: Add to Cart button with quantity stepper, Quick View modal sliding open, Cart drawer sliding from right edge. Items appear in cart with line totals.

**Narration:**
"The extension generates embed code for three commerce widgets. Add to Cart checks real-time inventory through the Shopify Storefront API. Quick View opens a focus-trapped modal with a mini gallery, variant picker, and instant cart add. And the Cart Manager persists in local storage — it survives page navigation, validates items on load, and redirects to Shopify checkout. All from copy-paste embeds."

---

## Scene 9 — Tag Reconciliation (2:55–3:15)

**Visual:** Tag cloud appears with duplicate clusters highlighted in red: "Leather" / "leather" / "LEATHER". Animation merges them into one. Blocklist terms fade out. Count badge updates.

**Narration:**
"Tags accumulate garbage across three systems. Mixed casing, trailing spaces, singular-plural duplicates. PIM Sync normalizes, deduplicates, and cleans tags automatically on every sync. You can set a blocklist for terms that should never appear, bulk-rename across all products, and preview the diff before applying. Clean data in, clean data out."

---

## Scene 10 — Three-Way Mirror (3:15–3:45)

**Visual:** Animated triangle diagram: Shopify at top, Xano center, Webflow bottom-right. Arrows pulse between them showing data flow directions. Echo detection markers appear and fade. Conflict flag appears, then resolves with "Keep Shopify" click.

**Narration:**
"The three-way mirror keeps Shopify, Xano, and Webflow synchronized. Change a product in Shopify — Xano picks it up via webhook, then pushes to Webflow. Edit in Webflow — it flows back to Xano, and the Shopify app handles the Admin API write. Echo detection prevents infinite loops — a 120-second KV marker and Xano state check ensure the same change doesn't bounce back. When both sides edit the same field, the conflict is flagged and you choose which version wins."

---

## Scene 11 — Non-Destructive Publish (3:45–4:10)

**Visual:** Checklist items appear and check off: "No deletes", "Draft-first", "Upsert only", "Field-level diff", "Price confirmation required". A green "Publish" button pulses. Sequence diagram shows Extension to Xano to Shopify App to Shopify store.

**Narration:**
"Publishing to a live Shopify store is high stakes. PIM Sync never deletes products — only upserts. New products land as drafts. Only changed fields are sent — unchanged data is never touched. Prices require explicit merchant confirmation. And the extension doesn't call the Shopify Admin API directly. It writes to Xano, and the PIM Sync Shopify app — already live in the App Store — handles all mutations. Your storefront stays safe."

---

## Scene 12 — Compliance and Security (4:10–4:25)

**Visual:** Shield icon with compliance badges appearing around it: GDPR, CCPA, PCI DSS, WCAG 2.2 AA, Omnibus. Lock icon with "TLS 1.2+", "HMAC", "API isolation". Fade to summary.

**Narration:**
"GDPR, CCPA, PCI DSS, EU Omnibus Directive, WCAG 2.2 AA — all covered. No customer data collected. No payment cards handled. All traffic over TLS. API groups isolated between PIM and CRM. Secrets rotatable independently. Every sync logged. Every webhook signature-verified."

---

## Scene 13 — Closing (4:25–4:35)

**Visual:** Three connected platform icons from Scene 3 return. Text appears: "PIM Sync — Webflow Designer Extension". Below: "Works standalone or paired with the PIM Sync Shopify App". App Store URL fades in.

**Narration:**
"PIM Sync. Install the Webflow extension. Connect your stack. Keep everything in sync."

---

## Post-Production Notes

- **Transitions:** Fade between scenes (0.3s). No slide animations — each scene builds in place.
- **Typography:** Inter or SF Pro. White text on dark (#0d1117) backgrounds. Accent blue (#58a6ff) for highlights.
- **Diagrams:** Flat design, monochrome with one accent color. No gradients, no 3D.
- **Music:** Recommend: Epidemic Sound "Ambient Tech" category, 90-110 BPM, fade under narration.
- **Captions:** Burned-in SRT captions, white text with dark background bar.
- **Thumbnail:** Hero shot of the 3-way triangle diagram with "PIM Sync" title, 1280x720.
