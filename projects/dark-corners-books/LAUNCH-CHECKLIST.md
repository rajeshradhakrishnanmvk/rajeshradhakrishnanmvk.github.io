# Shopify Store Launch Checklist — Dark Corners Books

## Phase 1: Foundation (Day 1-2)

### Account & Domain
- [ ] Create Shopify account (3-day free trial → $39/mo Basic plan)
- [ ] Register domain: `darkcornersbooks.com` (Shopify or Namecheap)
- [ ] Connect domain to Shopify
- [ ] Enable SSL (automatic with Shopify)

### Theme Setup
- [ ] Install Dawn theme (free, Shopify Theme Store)
- [ ] Go to Online Store → Themes → Customize
- [ ] Theme Settings → Colors:
  - Background: `#0a0a0a`
  - Text: `#e0d6c8`
  - Accent: `#b22222`
  - Button background: `#b22222`
- [ ] Theme Settings → Typography:
  - Headings: Georgia (serif)
  - Body: System font stack
- [ ] Theme Settings → Custom CSS: Paste ALL of `/theme/custom.css`
- [ ] Theme Settings → Favicon: Upload dark icon (red D on black)
- [ ] Theme Settings → Checkout: Upload checkout banner

### Store Settings
- [ ] Settings → Store Details:
  - Store name: Dark Corners Books
  - Legal name: [Your LLC/DBA]
  - Address: [Business address]
  - Email: hello@darkcornersbooks.com
- [ ] Settings → Payments:
  - Shopify Payments: Enable
  - PayPal Express: Enable
  - Manual: Disable
- [ ] Settings → Shipping:
  - Digital products: Create "Digital Delivery" profile — FREE shipping
  - Paperbacks: Create rate based on weight (via BookVault)
- [ ] Settings → Taxes:
  - Enable automatic tax calculation
  - Register for tax in your state/country
- [ ] Settings → Notifications:
  - Customize Order Confirmation email with dark theme
  - Add download instructions for ebooks

---

## Phase 2: Content (Day 2-3)

### Pages
- [ ] Create page: **Home** → Template: `page.homepage` → Copy from `/pages/homepage.md`
- [ ] Create page: **About** → Template: `page` → Copy from `/pages/about.md`
- [ ] Create page: **The Vault** → Template: `page.vault` → Copy from `/pages/the-vault.md`
- [ ] Create page: **Contact** → Simple form, email to hello@
- [ ] Create page: **Privacy Policy** → Shopify auto-generate
- [ ] Create page: **Terms of Service** → Shopify auto-generate
- [ ] Create page: **FAQ** → Shipping, formats, refunds

### Navigation
- [ ] Main Menu: Create using structure from `/config/navigation.md`
- [ ] Footer Menu: Create three-column footer menu
- [ ] Announcement Bar: "✦ Free horror story every month — Join The Vault ✦" → link to /pages/the-vault

### Collections
- [ ] Create: All Books (automatic, all products)
- [ ] Create: New Releases (tag: new-release)
- [ ] Create: Cosmic Horror (tag: cosmic-horror)
- [ ] Create: Folk Horror (tag: folk-horror)
- [ ] Create: Psychological Horror (tag: psychological-horror)
- [ ] Create: Bundles (product type: Bundle)
- [ ] Create: Ebooks (format filter)
- [ ] Create: Paperbacks (format filter)

---

## Phase 3: Products (Day 3)

### Add First Book
- [ ] Create product using template from `/products/product-template.md`
- [ ] Upload cover image (1600×2400px, 3:4 ratio, <5MB)
- [ ] Set up variants (Ebook / Paperback / Signed Paperback)
- [ ] Add product to relevant collections
- [ ] Add tags (genre, series, format, new-release)
- [ ] Set SEO title and description
- [ ] Test purchase flow (place a $0.01 test order)

### Digital Delivery Setup
- [ ] Install BookFunnel app from Shopify App Store
- [ ] Connect BookFunnel account
- [ ] Upload EPUB + PDF for each ebook product
- [ ] Configure auto-delivery on purchase
- [ ] Test: Buy ebook → Receive download email

### Print Fulfillment Setup
- [ ] Install BookVault app (or Lulu Direct)
- [ ] Connect account
- [ ] Set up paperback product with BookVault SKU mapping
- [ ] Configure shipping rates
- [ ] Test: Order paperback → BookVault processes

---

## Phase 4: Email & Marketing (Day 3-4)

### Klaviyo Setup
- [ ] Install Klaviyo from Shopify App Store
- [ ] Create list: "The Vault Subscribers"
- [ ] Create signup form → Embed on `/pages/the-vault`
- [ ] Create Welcome Flow (5 emails from `/email/welcome-sequence.md`)
- [ ] Set up abandoned cart flow (1 email, 4 hours after abandonment)
- [ ] Set up post-purchase flow (review request, 14 days after purchase)
- [ ] Test entire flow: Signup → Welcome sequence → Purchase → Post-purchase

### Popup / Signup Forms
- [ ] Klaviyo embedded form on The Vault page
- [ ] Footer newsletter block (all pages)
- [ ] Product page: "Want a free story?" banner → The Vault
- [ ] Post-purchase: "Join The Vault" checkbox (checked by default)

### Social Proof
- [ ] Install Judge.me or Loox for reviews
- [ ] Import existing Amazon/Goodreads reviews (manual)
- [ ] Set review request email (14 days post-purchase)

---

## Phase 5: Pre-Launch Testing (Day 4)

### Test Flows
- [ ] Mobile: Browse all pages, check responsive design
- [ ] Desktop: Same, all viewports
- [ ] Purchase ebook → Receive download
- [ ] Purchase paperback → Order routes to BookVault
- [ ] Sign up for The Vault → Receive email 1
- [ ] Wait 24h → Receive email 2 (automated test: trigger manually)
- [ ] Abandon cart → Receive abandoned cart email

### Speed Check
- [ ] Google PageSpeed Insights: Target 70+ mobile, 90+ desktop
- [ ] Shopify Speed Score: In Online Store → Themes
- [ ] Compress all images (Shopify auto-compresses, but verify)
- [ ] Check /pages/the-vault load time (high traffic page)

### Legal
- [ ] Privacy policy links in footer
- [ ] Terms of service links in footer
- [ ] Refund policy visible
- [ ] Cookie consent banner (Shopify built-in)
- [ ] Age gate on homepage (uses age-gate CSS class)
- [ ] GDPR compliance if selling to EU

---

## Phase 6: Launch (Day 5)

### Go Live
- [ ] Remove storefront password (Online Store → Preferences → Password protection → Disable)
- [ ] Submit sitemap to Google Search Console
- [ ] Verify domain in Google Search Console
- [ ] Enable Google Analytics (Shopify Settings)

### Launch Promotion
- [ ] Post on social media (Horror Booktok, Bookstagram, r/horrorlit)
- [ ] Email existing KDP readers (via back matter link)
- [ ] Run launch discount: "First 50 orders get 20% off" — code LAUNCH20

### Post-Launch (Week 1)
- [ ] Monitor orders daily
- [ ] Check Klaviyo flow performance
- [ ] Respond to all customer emails within 24 hours
- [ ] Fix any broken links or display issues

---

## Phase 7: Optimization (Ongoing)

### Weekly
- [ ] Check Shopify Analytics → Conversion rate (target: 2-4%)
- [ ] Review Klaviyo flows → Open rate, click rate, purchase rate
- [ ] Add new reviews as they come in

### Monthly
- [ ] Add new book products
- [ ] Update featured collection on homepage
- [ ] A/B test email subject lines
- [ ] A/B test bundle pricing
- [ ] Check PageSpeed scores

### Quarterly
- [ ] Review full P&L: Shopify fees, app costs, payment processing
- [ ] Evaluate Shopify plan (Basic $39 → Shopify $105 if >$5K/mo)
- [ ] Consider Shopify Markets for international sales
- [ ] Review BookVault/Lulu pricing vs alternatives
