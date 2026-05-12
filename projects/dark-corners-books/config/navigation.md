# Navigation Structure
# Shopify Admin → Online Store → Navigation

## Main Menu

```
DARK CORNERS BOOKS
├── New Releases          → /collections/new-releases
├── Cosmic Horror         → /collections/cosmic-horror
├── Folk Horror           → /collections/folk-horror
├── Psychological Horror  → /collections/psychological-horror
├── Bundles               → /collections/bundles
└── All Books             → /collections/all
```

## Footer Menu — Column 1: Shop

```
Shop
├── All Books             → /collections/all
├── New Releases          → /collections/new-releases
├── Bundles & Box Sets    → /collections/bundles
├── Ebooks                → /collections/ebooks
├── Paperbacks            → /collections/paperbacks
└── Gift Cards            → /products/gift-card
```

## Footer Menu — Column 2: The Vault

```
The Vault
├── Join Free             → /pages/the-vault
├── What Subscribers Get  → /pages/the-vault#perks
├── Reader Reviews        → /pages/the-vault#reviews
└── FAQ                   → /pages/the-vault#faq
```

## Footer Menu — Column 3: About

```
Dark Corners Books
├── Our Story             → /pages/about
├── About the Author      → /pages/about#author
├── Why Buy Direct        → /pages/about#why-direct
├── Contact               → /pages/contact
└── Rights & Licensing    → /pages/rights
```

## Sticky Announcement Bar

```
✦ Free horror story every month — Join The Vault ✦ [→]
```
Link: /pages/the-vault

---

## Collections to Create

### 1. All Books
**Handle:** `all`
**Conditions:** All products, automatically

### 2. New Releases
**Handle:** `new-releases`
**Conditions:** Tag = `new-release`
**Sort:** Newest first
**Products:** Manually tag 3-6 latest releases with `new-release`

### 3. Cosmic Horror
**Handle:** `cosmic-horror`
**Conditions:** Tag = `cosmic-horror`
**Description for SEO:**
> Cosmic horror fiction that reminds you how small you are. Lovecraftian terror, ancient entities, and the crushing weight of an indifferent universe. DRM-free ebooks and paperbacks.

### 4. Folk Horror
**Handle:** `folk-horror`
**Conditions:** Tag = `folk-horror`
**Description for SEO:**
> Folk horror rooted in ancient traditions. The old gods never left. The land remembers. The rituals continue. DRM-free ebooks and paperbacks drawing from global folklore.

### 5. Psychological Horror
**Handle:** `psychological-horror`
**Conditions:** Tag = `psychological-horror`
**Description for SEO:**
> Psychological horror that gets inside your head. The monster isn't under the bed — it's in your mind, and it's been there a long time. DRM-free ebooks and paperbacks.

### 6. Bundles
**Handle:** `bundles`
**Conditions:** Product type = `Bundle`
**Description for SEO:**
> Complete horror series at bundle pricing. Save 40-50% vs individual purchases. DRM-free ebooks delivered instantly.

### 7. Ebooks
**Handle:** `ebooks`
**Conditions:** Type = `Ebook` OR tag = `ebook`

### 8. Paperbacks
**Handle:** `paperbacks`
**Conditions:** Type = `Paperback` OR tag = `paperback`
