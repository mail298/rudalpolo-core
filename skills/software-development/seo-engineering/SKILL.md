---
name: seo-engineering
description: "Use when optimizing web apps or sites for search engines."
version: 1.0.0
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [seo, technical-seo, semantic-html, open-graph, json-ld, structured-data, core-web-vitals, geo]
---

# Production SEO Engineering Standards

## Overview

Modern SEO for engineering teams is not superficial keyword stuffing. It is an end-to-end technical discipline that spans **semantic HTML architecture**, **meta tag standards**, **social graph previews (Open Graph & Twitter Cards)**, **machine-readable structured data (Schema.org JSON-LD)**, **crawl budget & indexing controls (robots.txt, sitemap.xml)**, and modern **Generative Engine Optimization (GEO)** for AI crawlers (ChatGPT, Perplexity, Claude, Google AI Overviews).

---

## Core Engineering Workflows

### 1. Semantic HTML & Document Architecture

#### The Golden Hierarchy Rule
- **Single `<h1>` Per Page:** Exactly one primary heading expressing the distinct purpose of the page.
- **Strict Heading Cascading:** Never jump from `<h1>` directly to `<h3>`. Maintain logical hierarchy (`h1` -> `h2` -> `h3`).
- **Semantic Tags Over `<div>` Sprawl:**
  - `<header>` for navigation and branding.
  - `<main>` for primary unique page content.
  - `<article>` for self-contained cards, posts, or standalone items.
  - `<nav>` for navigation links.
  - `<footer>` for metadata, copyrights, and secondary links.
- **Accessible Images & Media:**
  - Every `<img>` must have an informative `alt` attribute. If purely decorative, use `alt=""` and `aria-hidden="true"`.
  - Provide explicit `width` and `height` (or aspect-ratio) to prevent Cumulative Layout Shift (CLS).

---

### 2. High-CTR Meta Tags & Social Graphs

Every production page must include the following in `<head>`:

```html
<!-- Primary Meta Tags -->
<title>Noted_R — Sticky Notes Minimalis, Cepat, & Bebas Ribet</title>
<meta name="title" content="Noted_R — Sticky Notes Minimalis, Cepat, & Bebas Ribet" />
<meta name="description" content="Aplikasi catatan tempel digital multi-warna super cepat, zero-friction, auto-save real-time, dan sinkronisasi cloud lintas perangkat dengan proteksi PIN rahasia." />
<meta name="keywords" content="sticky notes, catatan tempel, quick notes, memo online, cloud sync notes, noted_r" />
<meta name="author" content="Windri" />
<meta name="robots" content="index, follow" />
<link rel="canonical" href="https://check.wn-cdn.net/noted-r/" />

<!-- Open Graph / Facebook / WhatsApp -->
<meta property="og:type" content="website" />
<meta property="og:url" content="https://check.wn-cdn.net/noted-r/" />
<meta property="og:title" content="Noted_R — Sticky Notes Minimalis & Cepat" />
<meta property="og:description" content="Catatan tempel modern multi-warna, auto-save instan, dan sinkronisasi cloud lintas perangkat." />
<meta property="og:image" content="https://check.wn-cdn.net/noted-r/og-preview.png" />
<meta property="og:site_name" content="Noted_R" />

<!-- Twitter / X -->
<meta property="twitter:card" content="summary_large_image" />
<meta property="twitter:url" content="https://check.wn-cdn.net/noted-r/" />
<meta property="twitter:title" content="Noted_R — Sticky Notes Minimalis & Cepat" />
<meta property="twitter:description" content="Catatan tempel modern multi-warna, auto-save instan, dan sinkronisasi cloud lintas perangkat." />
<meta property="twitter:image" content="https://check.wn-cdn.net/noted-r/og-preview.png" />
```

---

### 3. Structured Data (Schema.org / JSON-LD)

Structured data enables Google Rich Results (software rating, price, category, FAQ). Embed via `<script type="application/ld+json">` in the HTML document:

#### SoftwareApplication / WebApplication Schema
```json
{
  "@context": "https://schema.org",
  "@type": "WebApplication",
  "name": "Noted_R",
  "url": "https://check.wn-cdn.net/noted-r/",
  "applicationCategory": "ProductivityApplication",
  "operatingSystem": "All",
  "browserRequirements": "Requires JavaScript. Requires HTML5.",
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "USD"
  },
  "description": "Aplikasi catatan tempel digital minimalis dengan auto-save instan, multi-warna, dan sinkronisasi cloud otomatis.",
  "author": {
    "@type": "Person",
    "name": "Windri"
  }
}
```

---

### 4. Crawl Control: `robots.txt` & `sitemap.xml`

#### Production `robots.txt`
Place at domain root `/robots.txt`:
```txt
User-agent: *
Allow: /
Disallow: /api/
Disallow: /storage/

Sitemap: https://check.wn-cdn.net/noted-r/sitemap.xml
```

#### Production `sitemap.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://check.wn-cdn.net/noted-r/</loc>
    <lastmod>2026-09-13</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```

---

### 5. GEO (Generative Engine Optimization) & `llms.txt`

To ensure modern AI engines (ChatGPT, Perplexity, Claude, Google Gemini) accurately quote and understand your application, provide an `/llms.txt` file at the root:

```markdown
# Noted_R (Sticky Notes)

> Aplikasi catatan tempel digital minimalis, zero-friction, multi-warna, dan tersinkronisasi cloud.

## Overview
Noted_R dirancang untuk menangkap ide, to-do list, dan memo seketika tanpa rintangan navigasi atau form yang rumit. 

## Key Features
- Zero-Friction Capture: Buka langsung ketik, auto-save instan pada setiap ketukan keyboard.
- Microsoft Sticky Notes WYSIWYG formatting (Bold, Italic, Underline, Strikethrough, Bullet, Checklist).
- Stealth PIN Lock: Perlindungan privasi catatan dengan gembok PIN tersinkronisasi cloud.
- Ultra-Wide Responsive Maximize: Tampilan fokus layar penuh adaptif untuk HP, tablet, dan monitor ultra-wide.
- Real Cross-Device Cloud Sync: Sinkronisasi database SQLite otomatis antar perangkat.
```

---

### 6. Single Page App (SPA) Crawlability Pitfall

#### The Pitfall
Pure client-side SPAs render only `<div id="root"></div>`. While modern Googlebot executes JavaScript, other crawlers (social media bots, WhatsApp/Telegram link scrapers, slower search engines) do not run JS and see an empty page.

#### Mandatory Rules
1. **Static Fallback in HTML Body:**
   Always place basic semantic branding, title, and descriptive summary inside `<div id="root">` or a `<noscript>` block inside `index.html`. When JS loads, React will replace it, but crawlers without JS get immediate readable content.
2. **Dynamic Meta via Header / Static HTML:**
   Place essential Open Graph, Twitter, and canonical tags directly in static `index.html` rather than relying solely on `react-helmet` or client JS.
