# Dark Corners Books — Shopify Store Package

Complete Shopify store structure for a horror fiction direct-sales business. Dark theme, Dawn-optimized, with full content, email sequences, and launch playbook.

---

## What's Included

```
horror-bookstore/
├── README.md                    ← You are here
├── LAUNCH-CHECKLIST.md          ← 7-phase launch playbook
│
├── theme/
│   └── custom.css               ← Full dark horror theme (paste into Dawn)
│
├── config/
│   ├── theme-settings.json      ← Colors, fonts, layout reference
│   └── navigation.md            ← Menu structure + collection setup
│
├── pages/
│   ├── homepage.md              ← Full homepage content + sections
│   ├── about.md                 ← Author + publisher story
│   └── the-vault.md             ← Email signup landing page
│
├── products/
│   ├── product-template.md      ← Book product page template + back matter
│   └── sample-books.md          ← 4 sample products (copy + adapt)
│
├── email/
│   └── welcome-sequence.md      ← 5-email Klaviyo welcome flow
│
└── assets/
    └── (cover templates, Midjourney prompts)
```

---

## Quick Start (30 Minutes to Live Store)

### 1. Create Shopify Account (5 min)
- Go to shopify.com → Start free trial
- Choose store name: **Dark Corners Books**
- Skip the setup wizard (we'll do it manually)

### 2. Apply Theme (10 min)
1. Online Store → Themes → Customize (Dawn)
2. Theme Settings → Colors → Set from `config/theme-settings.json`
3. Theme Settings → Typography → Set from `config/theme-settings.json`
4. Theme Settings → Custom CSS → Paste ALL of `theme/custom.css`
5. Save

### 3. Create Pages (10 min)
1. Online Store → Pages → Add Page
2. Copy content from `pages/homepage.md`, `pages/about.md`, `pages/the-vault.md`
3. Set homepage as front page: Online Store → Preferences → Homepage

### 4. Set Navigation (5 min)
1. Online Store → Navigation → Main Menu → Follow `config/navigation.md`
2. Footer Menu → Same file, footer columns

### 5. Add Products (varies)
- Copy templates from `products/product-template.md` and `products/sample-books.md`
- Upload covers, set prices, connect to collections

### 6. Install Apps
- **BookFunnel** — Digital delivery
- **BookVault** or **Lulu Direct** — Print fulfillment
- **Klaviyo** — Email marketing
- **Judge.me** — Reviews

### 7. Import Email Flows
- Klaviyo → Flows → Create Flow → Copy from `email/welcome-sequence.md`
- 5 emails over 5 days

### 8. Launch
- Remove password protection
- Follow `LAUNCH-CHECKLIST.md` Phase 6

---

## Monthly Costs (Basic Plan)

| Item | Cost |
|------|------|
| Shopify Basic | $39/mo |
| Domain (annual) | ~$15/year |
| BookFunnel | $20/mo |
| BookVault | Per-book printing |
| Klaviyo | Free (up to 250 contacts) → $30/mo at 1K+ |
| Judge.me | Free |
| **Total** | **~$75-100/mo** |

---

## Key Design Decisions

1. **Dark theme** — Horror readers expect it. Matches genre conventions.
2. **Dawn theme** — Shopify's fastest free theme. Good for SEO.
3. **No homepage slider** — Sliders hurt conversion. Fixed hero.
4. **Age gate** — Optional but recommended for horror. CSS included.
5. **The Vault first** — Email capture prioritized over immediate sale.
6. **Shopify links first in back matter** — When given dual links (Amazon + Shopify), Shopify is always listed first in book back matter to prioritize higher-margin sales.

---

## Integration with KDP

This store works alongside, not instead of, Amazon KDP:

```
Amazon KDP                    Shopify Store
────────────                  ─────────────
Discovery (300M+)             Profit (95% margin)
Kindle Unlimited reads        Direct customer relationship
Amazon Ads                    Email list ownership
Reviews on Amazon             Reviews on Judge.me
                              ↓                    ↓
                    Back matter in every book
                    "Join The Vault" → Shopify signup
                              ↓
                    Email Welcome Sequence
                              ↓
                    Shopify Direct Sales
```

Do NOT enroll Shopify-sold ebooks in KDP Select (requires exclusivity). Either:
- **Option A (recommended for new authors):** Enroll in KDP Select for 90 days → build audience → pull out → sell direct on Shopify
- **Option B (for established authors):** Skip KDP Select. Sell wide (KDP + Shopify + Google Play + Kobo)

---

## Horror-Specific Notes

- **Cover prices:** $2.99 is the sweet spot for horror novellas (70% KDP royalty threshold)
- **Bundle pricing:** 40-50% off individual prices. Horror readers binge and respond to bundles.
- **Seasonal promotion:** Double marketing spend in October. Halloween-themed bundles.
- **Reader magnet:** The free short story ("The Waking Dark") is your primary list-builder. Make it great.
- **Tone:** Atmospheric, not gory. Sell the dread, not the body count.

---

## Next Steps After Store is Live

1. Publish first horror novella on KDP
2. Add "Join The Vault" link in back matter
3. Run $5/day Amazon Sponsored Products ads
4. When email list hits 200 → Send first Shopify-exclusive bundle offer
5. When revenue >$5K/mo → Upgrade to Shopify plan ($105/mo, lower fees)
6. When list >2K → Pull bestsellers from KU, sell wide
