# SEO / AEO Preservation Requirements — iloveahandyman.com

**This file OVERRIDES the `/clone-website` skill's default scope.** The skill lists
"SEO optimization" as *out of scope* and defaults to cloning only the single page the
target URL resolves to. Both defaults are **void for this run.** The client's stated
goal is: *retain 100% of existing organic traffic through the migration, and improve on
it where possible.*

Treat every item below as a build requirement, not a nice-to-have. A pixel-perfect clone
that loses rankings is a failed clone.

---

## Client decisions (confirmed 2026-09-09)

| Decision | Answer | Consequence for the build |
| --- | --- | --- |
| Domain plan | **Replace the live site.** `iloveahandyman.com` will point at the new build; the WordPress site is retired. | Highest-risk path. Full URL parity is **mandatory**, not optional. Redirect map and the §9 cutover checklist are required deliverables. |
| Google Search Console | **Not available yet** — site owner doesn't have access to hand over. Build proceeds without it. | URL inventory comes from `sitemap.xml` + full internal-link crawl (§0 items 1–3). **GSC access is a hard gate before DNS cutover** — see below. |
| Push target | **Straight to `main`.** | Lovable auto-syncs each push. No review branch. Only push after `bun run build` passes and QA is done. |

### Deferred, but blocking at cutover

GSC is deferred for the build phase only. **Do not cut over DNS without it.** Without
Search Console the client cannot:

- capture the pre-migration traffic baseline (§9 step 7) — with no baseline, a drop is
  undetectable and unprovable
- submit the new sitemap or request re-indexing, which is what makes Google recrawl
  quickly instead of over weeks
- see crawl errors, coverage loss, or manual actions after the switch

It is free and takes minutes, but **verification must be added to the live WordPress site
while it is still up** — the DNS TXT or HTML-tag verification is far easier to complete
before the switch than after. Raise this with the client well ahead of cutover, and carry
any existing verification meta tag across per §3.

Because the site owner may not have GSC set up at all, also confirm who controls the
**Google Business Profile** — for a local handyman business that listing likely drives
more calls than organic search does, and the NAP consistency requirement in §4 depends on
being able to read it.

---

## 0. The single biggest risk: partial-site clones

If the live site has N indexed pages and we clone only the homepage, then point the
domain at the new build, **every other ranking URL becomes a 404 and that traffic is
gone.** Recovering it takes months, if it comes back at all.

**Requirement:** before building anything, enumerate the full set of indexed URLs and
clone *all* of them. Sources, in order of reliability:

1. `https://iloveahandyman.com/sitemap.xml` (also `/sitemap_index.xml`, `/wp-sitemap.xml`
   — WordPress/Yoast/RankMath variants)
2. `robots.txt` (often names the sitemap)
3. Crawl every internal link from the homepage, recursively, same-origin only
4. Google Search Console → Pages report (client-supplied; the authoritative list of what
   actually ranks and gets clicks)

Write the result to `URL_INVENTORY.md` in this folder: every URL, its title, its meta
description, and — where GSC data is available — its clicks/impressions. **Any URL with
meaningful traffic is mandatory scope.** Confirm the final page list with the client
before the build begins.

---

## 1. URL structure — exact parity

- Every existing path must resolve at the **identical path** in the clone. No renaming,
  no restructuring, no "cleaner" URLs.
- **Trailing slashes are a real risk here.** WordPress serves `/services/` *with* a
  trailing slash; TanStack Router's `createRouter` in `src/router.tsx` does not currently
  set `trailingSlash`, so its default applies and will likely mismatch. Determine the
  live site's convention, then set `trailingSlash` explicitly to match, and verify a
  request to the original form returns **200, not a 301 or 404**.
- Preserve the `www` vs non-`www` choice and the `http → https` upgrade exactly as the
  live site resolves them today.
- Case sensitivity: WordPress is typically case-insensitive on paths; the new stack is
  not. Check for any capitalized URLs in the inventory.

**If any URL genuinely must change,** it requires a **301** (permanent) redirect from old
to new — never a 302, never a client-side redirect, never a JS `router.navigate`. Search
engines only pass ranking signals through a server-side 301.

---

## 2. Semantic HTML — the silent clone-killer

The extraction pipeline works from `getComputedStyle()`, which describes *appearance*,
not *meaning*. A builder agent handed "font-size: 40px; font-weight: 700" will happily
emit a `<div>`. That renders identically and is **catastrophic for SEO.**

**Requirement — every component spec and every builder prompt must state:**

- The real tag for every text node: `<h1>`, `<h2>`, `<h3>`, `<p>`, `<ul>`/`<li>`,
  `<nav>`, `<main>`, `<header>`, `<footer>`, `<article>`, `<section>`.
- Exactly **one `<h1>` per page**, carrying the same text as the live page's `<h1>`.
- Heading hierarchy preserved in order and depth — no skipped levels, no visual-only
  headings.
- Every `<img>` keeps its **verbatim `alt` text**. Empty `alt=""` on decorative images
  stays empty — that is a deliberate signal, not an omission.
- Internal links keep their **exact anchor text** and use real `<a>`/`<Link>` elements.
  Anchor text is a ranking signal; "click here" is not a valid substitute.
- Preserve `<title>` and heading text **verbatim**, including keyword phrasing that may
  look awkward. Do not "improve" copy during the emulation phase.

Add a per-page extraction artifact recording the full heading outline and every
`alt`/anchor string, so the built page can be diffed against it.

---

## 3. Meta parity — per page, not site-wide

The Lovable scaffold ships placeholder meta in `src/routes/__root.tsx` that **must not
ship to production**: `"Lovable App"`, `"Lovable Generated Project"`,
`author: "Lovable"`, `twitter:site: "@Lovable"`. Replace all of them.

For **each** route, extract from the live page and reproduce via that route's `head()`:

| Tag | Notes |
| --- | --- |
| `<title>` | verbatim, per page |
| `meta[name=description]` | verbatim, per page |
| `link[rel=canonical]` | **self-referencing absolute URL**, per page — the scaffold has none by default |
| `meta[property=og:*]` | title, description, image, url, type, site_name |
| `meta[name=twitter:*]` | card, title, description, image |
| `meta[name=robots]` | copy `noindex`/`nofollow` exactly where present — do not "fix" it |
| `link[rel=alternate]` | hreflang, RSS, if present |
| Site verification tags | Google, Bing, Facebook domain verification — **losing these breaks GSC access** |

Canonical tags need an absolute URL built from the production origin. Do not hardcode a
Lovable preview domain into canonicals — that would tell Google the preview is the real
site.

---

## 4. Structured data (JSON-LD) — this is the AEO layer

For a local service business, structured data is what drives Google's local pack, rich
results, and AI/answer-engine citations. This is the highest-leverage item for the
"make it better" half of the goal.

**Preserve** every JSON-LD block on the live site verbatim first (extract from
`script[type="application/ld+json"]`). WordPress SEO plugins emit these automatically and
they will vanish the moment we leave WordPress.

**Then verify these exist and are complete** — the likely wins:

- `LocalBusiness` (or `HomeAndConstructionBusiness` / `GeneralContractor`) with exact
  **NAP**: business name, full postal address, phone. NAP must match Google Business
  Profile **character for character** — inconsistency actively suppresses local ranking.
- `geo` coordinates, `areaServed`, `openingHoursSpecification`, `priceRange`
- `sameAs` links to every social/GBP profile
- `Service` entries per service offered
- `AggregateRating` / `Review` — **only if genuine reviews exist on the page.** Never
  fabricate ratings; Google penalizes it and it is a real legal exposure.
- `FAQPage` where the page has genuine Q&A content
- `BreadcrumbList` matching the visual breadcrumbs

In TanStack Start, JSON-LD goes through the route's `head()` (verify the installed
version's support for a `scripts` entry with `type: "application/ld+json"`; if
unsupported, render the `<script>` in the route component — it must be **server-rendered
and present in view-source**, not injected client-side).

Validate every page against Google's Rich Results Test before cutover.

---

## 5. robots.txt and sitemap.xml

- `public/robots.txt` currently carries the Lovable default and **names no sitemap.**
  Add `Sitemap: https://<production-domain>/sitemap.xml`, and mirror any `Disallow` rules
  from the live site.
- Generate a real `sitemap.xml` covering every cloned URL with accurate `<lastmod>`.
  Static in `public/` is acceptable for a fixed page set; if pages become dynamic, serve
  it from a server route instead.
- Confirm the built output serves both at the site root with `content-type` correct.

---

## 6. Status codes and crawlability

Server-rendered output is the whole point — verify with `curl`, not the browser:

- Every cloned URL returns **200** with **fully rendered HTML in the response body**
  (`curl -s <url> | grep "<h1"`). If content only appears after hydration, it is invisible
  to some crawlers.
- Unknown URLs return a real **404** status, not 200-with-404-content ("soft 404").
  Verify `notFoundComponent` in `__root.tsx` produces an actual 404 status code.
- No `noindex` leaks into production — check the built HTML for stray robots meta.
- The Lovable preview domain should **not** be indexable; only the production domain.

---

## 7. Analytics and tracking continuity

Extract and carry over, or the client loses measurement continuity at the exact moment
they most need it:

- Google Analytics / GA4 measurement ID
- Google Tag Manager container
- Google Search Console verification
- Meta/Facebook Pixel, call-tracking scripts, chat widgets
- Any conversion tracking tied to form submissions or phone-number clicks

**Note:** the live site's contact form is almost certainly a WordPress plugin with a
server-side mail handler. A static clone will render the form but **silently stop
delivering leads.** This is a business-critical failure mode independent of SEO — flag it
to the client and agree on a replacement (embed the original form provider, or wire a
real handler) before cutover.

---

## 8. Performance — the improvement opportunity

SSR + Tailwind v4 should beat WordPress + Elementor comfortably. To realize it:

- Serve images in modern formats at correct dimensions; set explicit `width`/`height` on
  every `<img>` to eliminate layout shift (CLS).
- `loading="lazy"` below the fold; **eager** for the hero/LCP image, with `fetchpriority="high"`.
- Self-host fonts or preconnect to `fonts.gstatic.com`; use `font-display: swap`.
- Measure before and after with Lighthouse and record both in this folder. Core Web
  Vitals are a live ranking input.

---

## 9. Cutover checklist (run in this order)

SEO is won or lost here, not in the code.

1. Full-site clone complete, every URL in `URL_INVENTORY.md` verified 200 + correct meta
2. Crawl the staging build (Screaming Frog or equivalent) and **diff against a crawl of
   the live site**: titles, descriptions, H1s, canonicals, status codes. Zero unexplained deltas.
3. Rich Results Test passes on every page type
4. Redirect map in place and tested for any changed URL
5. Analytics + GSC verification confirmed present in built HTML
6. Contact form tested end-to-end — a real submission arrives
7. **Take a full GSC + GA export immediately before cutover** as the performance baseline
8. Lower DNS TTL ~24h ahead of the switch
9. Cut over DNS
10. In GSC: submit the new sitemap, request indexing on top pages, watch Coverage daily
11. Monitor rankings and traffic daily for 2 weeks, weekly for 8. Expect a small,
    temporary dip; investigate anything beyond ~10% lasting more than 2 weeks.
12. Keep the old WordPress site backed up and restorable for at least 90 days.

---

## 10. Honest expectations

A same-domain, same-URL, same-content migration to a faster stack typically holds
rankings, with brief volatility while Google recrawls. It is not automatic: it depends on
the parity above being genuinely complete.

Gains beyond parity come from speed, cleaner semantics, and richer structured data — real
but incremental, and they arrive over weeks. Anyone promising a traffic increase on a
fixed schedule is guessing. The reliable outcome here is **hold the traffic, then compound
from a faster, better-marked-up base.**
