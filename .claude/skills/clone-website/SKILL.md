---
name: clone-website
description: Reverse-engineer and clone one or more websites in one shot — extracts assets, CSS, and content section-by-section and proactively dispatches parallel builder agents in worktrees as it goes. Output is a TanStack Start (SSR) + React 19 + Tailwind v4 codebase that matches Lovable's current scaffold and ports into a Lovable-created GitHub repo (Lovable cannot import external repos). Use this whenever the user wants to clone, replicate, rebuild, reverse-engineer, or copy any website. Also triggers on phrases like "make a copy of this site", "rebuild this page", "pixel-perfect clone". Provide the target site URL(s) plus the Lovable repo URL (created via Lovable → Connect to GitHub) as arguments.
argument-hint: "<target-url> [<target-url2> ...] <lovable-repo-url>"
user-invocable: true
---

# Clone Website

You are about to reverse-engineer and rebuild the **target site(s)** passed in `$ARGUMENTS` as pixel-perfect clones, then port the result into the user's **Lovable repo** (also passed in `$ARGUMENTS`). Pre-Flight explains how the arguments are split — the GitHub URL is the Lovable build target; every other URL is a site to clone.

This template repo is reused for **every clone the user ever runs** — one site today, a different one next week. So all extraction artifacts are **always** isolated per site in hostname folders (`docs/research/<hostname>/`, `docs/design-references/<hostname>/`), even for a single-target run. When multiple target URLs are provided, process them independently and in parallel where possible.

This is not a two-phase process (inspect then build). You are a **foreman walking the job site** — as you inspect each section of the page, you write a detailed specification to a file, then hand that file to a specialist builder agent with everything they need. Extraction and construction happen in parallel, but extraction is meticulous and produces auditable artifacts.

## Scope Defaults

The target is whatever page `$ARGUMENTS` resolves to. Clone exactly what's visible at that URL. Unless the user specifies otherwise, use these defaults:

- **Fidelity level:** Pixel-perfect — exact match in colors, spacing, typography, animations
- **In scope:** Visual layout and styling, component structure and interactions, responsive design, mock data for demo purposes
- **Out of scope:** Real backend / database, authentication, real-time features, SEO optimization, accessibility audit
- **Output:** A frontend clone built on Lovable's current scaffold — **TanStack Start (SSR) + React 19 + Tailwind v4**. SSR is the target, not something to avoid. The finished repo must match Lovable's scaffold so it can be ported into a Lovable-created repo (see Phase 6) — the constraint is: don't introduce a foreign framework, don't add a second `package.json`, and don't strip out Lovable's scaffold files
- **Customization:** None — pure emulation

If the user provides additional instructions (specific fidelity level, customizations, extra context), honor those over the defaults.

## Target Stack

The deliverable targets Lovable's **live scaffold**, which (as of Aug 2026) is:

- **Framework:** `@tanstack/react-start` + `@tanstack/react-router` (file-based routing), SSR via **nitro** (default build target: cloudflare). React **19**, TypeScript.
- **Styling:** Tailwind **v4** (`@tailwindcss/vite`, `tw-animate-css`) + shadcn/ui "new-york". The Tailwind entry is **`src/styles.css`** (imported in `src/routes/__root.tsx` as `import appCss from "../styles.css?url"`). There is NO `tailwind.config.ts` and NO `postcss.config` — v4 config lives in `styles.css` via `@import "tailwindcss" source(none); @source "../src"; @theme inline { ... }` plus `:root`/`.dark` oklch tokens.
- **No `index.html`.** The HTML document shell is `src/routes/__root.tsx` (`RootShell` renders `<html><head><HeadContent/></head><body>{children}<Scripts/></body></html>`). Head/meta/links/favicon/fonts are set via a route's `head()` export returning `{ meta: [...], links: [...] }` (TanStack merges root + per-route head).
- **Pages = routes.** The home page is **`src/routes/index.tsx`** via `export const Route = createFileRoute("/")({ head: () => ({...}), component: Index })`. There is NO `src/pages/`, NO `src/App.tsx`, NO `src/main.tsx`, NO `<BrowserRouter>`, NO `react-router-dom`. Additional cloned pages = additional files in `src/routes/` (file-based routing regenerates `routeTree.gen.ts` at build). Internal links use `<Link>` from `@tanstack/react-router`.
- **`@/` alias → `src/`.** `cn()` in `src/lib/utils.ts` and shadcn primitives in `src/components/ui/` are unchanged from the old scaffold.
- **Package manager / build:** the repo uses **bun** (`bun.lock`, `bunfig.toml`). Build/typecheck with `bun run build` (vite build + nitro; typechecks via the bundler). There is no standalone `typecheck` script — use `bun run build` and/or `npx tsc --noEmit`.
- **Lovable-specific files that MUST be preserved:** `.lovable/project.json`, `src/lib/lovable-error-reporting.ts`, `src/server.ts` (SSR error wrapper), `src/start.ts`, `src/router.tsx`, `src/routes/__root.tsx`, `vite.config.ts` (uses `@lovable.dev/vite-tanstack-config`, which bundles tanstackStart/viteReact/tailwind/tsconfigPaths/nitro — do NOT add these plugins manually), `package.json`.

Because Lovable's stack evolves, confirm the scaffold shape from an **actual freshly-created Lovable repo** rather than assuming these exact facts. **Fallback:** if the user only wants a standalone site (not destined for Lovable), a plain Vite React SPA is still fine — but the Lovable path requires matching Lovable's scaffold.

### SSR gotchas

Because the page is server-rendered and then hydrated:

- **No `window` / `document` / `localStorage` at module scope or during render.** Guard all browser APIs inside `useEffect` (client-only). `IntersectionObserver`, `matchMedia`, etc. go in effects.
- **Time-based / random UI causes hydration mismatches.** For live countdowns, "spots remaining" randomizers, or anything using `Date.now()` / `Math.random()` in the initial render, add `suppressHydrationWarning` on the specific element, or defer with a mounted flag (`const [m, setM] = useState(false); useEffect(() => setM(true), [])`).
- Components are plain React function components (React 19 + SSR). **No `"use client"` directives** — that's Next.js, not TanStack Start.
- Assets still resolve from `public/` at web root (`/images/...`, `/seo/...`).

## Pre-Flight

Your session's workspace is **this template repo** — it holds the skill, the headless-browser config (`.mcp.json`), and the docs structure. **It has no app to build.** The clone is built inside a **separate clone of the user's Lovable repo**, which is the real build target. Work through every check below *before* extracting anything — each one catches a mistake that otherwise only surfaces an hour into a clone, after real work is wasted. Write failures in plain English for a non-technical student.

1. **Browser automation is required.** This repo ships a headless Playwright MCP server preconfigured in `.mcp.json` — it works identically in Claude Code on the web (cloud sandbox) and in local/desktop sessions, so prefer it for reproducibility. If its browser is missing, install it first: `npx playwright install chromium` (add `--with-deps` if the environment supports apt). Other browser MCP tools (Chrome MCP, Browserbase, Puppeteer) are acceptable in local sessions if the user prefers them. If no browser tool can start, tell the user: in a cloud session this usually means the environment's network access is too restricted — it must allow reaching the target site and downloading the Playwright browser. This skill cannot work without browser automation.

2. **Split the arguments into target URL(s) + the Lovable repo URL.** `$ARGUMENTS` contains one or more **target site URLs** to clone, plus **one Lovable repo URL** (the build target — a `https://github.com/<owner>/<repo>.git` the user created *through Lovable*). The GitHub URL among the arguments is the Lovable repo; every other URL is a target site.
   - **If no Lovable repo URL was provided, STOP and ask for it** in plain language, e.g.: *"Which GitHub repo should I build your clone into? In Lovable: create a project → open the GitHub menu → Connect to GitHub → Create Repository, then copy that repo's URL and paste it here."* Do **not** fall back to building in this template — the template is not the deliverable and Lovable can't import it.
   - Normalize and validate each target URL; if any are malformed, ask the user to fix them.

3. **Confirm network access + target reachability.** Load each target URL in the browser MCP. If the page won't load, tell the student plainly: *"I can't reach the site (or the internet). In a cloud Claude Code session, open the environment/network settings and turn on network access, then ask me to retry — this is the #1 reason a clone stalls at the very start."* Don't continue until every target loads.

4. **Verify the Lovable repo is clonable AND pushable — now, before any clone work.** Pushing to the repo's synced branch is the ONLY way the finished clone reaches Lovable, and a student's GitHub connection often can't push to it. Catch that here, not after an hour.
   - **Clone it** next to (not inside) this template: `git clone <lovable-repo-url>` into a sibling working directory. If the clone fails with an auth/permission error, the session's GitHub connection can't even read the repo — send the student to the fix below.
   - **Confirm push access without writing anything:** `git -C <clone-dir> push --dry-run origin HEAD`. If it fails with a permission/auth error, **STOP** and tell the student, plainly:
     > *"I can read your Lovable repo but I can't push to it yet — so I'd be able to build your clone but not deliver it. Claude Code's GitHub connection usually only has access to this template repo, not your new Lovable repo. Fix it once: GitHub → your avatar → **Settings → Applications → Claude Code (GitHub App) → Configure** → under **Repository access** add your Lovable repo (or pick **All repositories**) → **Save**. Then tell me to retry."*
   - Continue only once **both** the clone and `push --dry-run` succeed.

5. **Confirm the scaffold from the REAL repo — don't assume the stack.** Read the freshly cloned Lovable repo and confirm what it actually is: framework, where global styles live, how the document `<head>` is set, how routing/pages work, which files are Lovable's own. The **Target Stack** section documents what Lovable ships *as of Aug 2026* (TanStack Start / SSR / React 19 / Tailwind v4) — treat that as a strong prior, **not** gospel. If what you find differs (Lovable changed stacks again, as it has before), **adapt to what's actually there** rather than forcing the documented structure or failing. Then verify the untouched scaffold builds: `bun install` then `bun run build`. **This cloned Lovable repo is your build workspace for everything that follows** — all later references to "the project" mean this repo.

6. Create the extraction-artifact directories in **this template** for the current site: `docs/research/<hostname>/`, `docs/research/<hostname>/components/`, `docs/design-references/<hostname>/`, and `scripts/<hostname>/`. **Always use the per-hostname folders** — this template is reused across every clone the user runs, so artifacts from different sites must never mix, and a folder that already exists for this hostname means a previous run: review what's there before overwriting. Every later mention of `docs/research/...`, `docs/design-references/...`, or `scripts/...` in this skill means the path **inside the current site's hostname folder**. (Specs and screenshots live here in the template; built code and assets go into the cloned Lovable repo.)
7. When working with multiple sites in one command, optionally confirm whether to run them in parallel (recommended, if resources allow) or sequentially to avoid overload.

## Guiding Principles

These are the truths that separate a successful clone from a "close enough" mess. Internalize them — they should inform every decision you make.

### 1. Completeness Beats Speed

Every builder agent must receive **everything** it needs to do its job perfectly: screenshot, exact CSS values, downloaded assets with local paths, real text content, component structure. If a builder has to guess anything — a color, a font size, a padding value — you have failed at extraction. Take the extra minute to extract one more property rather than shipping an incomplete brief.

### 2. Small Tasks, Perfect Results

When an agent gets "build the entire features section," it glosses over details — it approximates spacing, guesses font sizes, and produces something "close enough" but clearly wrong. When it gets a single focused component with exact CSS values, it nails it every time.

Look at each section and judge its complexity. A simple banner with a heading and a button? One agent. A complex section with 3 different card variants, each with unique hover states and internal layouts? One agent per card variant plus one for the section wrapper. When in doubt, make it smaller.

**Complexity budget rule:** If a builder prompt exceeds ~150 lines of spec content, the section is too complex for one agent. Break it into smaller pieces. This is a mechanical check — don't override it with "but it's all related."

### 3. Real Content, Real Assets

Extract the actual text, images, videos, and SVGs from the live site. This is a clone, not a mockup. Use `element.textContent`, download every `<img>` and `<video>`, extract inline `<svg>` elements as React components. The only time you generate content is when something is clearly server-generated and unique per session.

**Layered assets matter.** A section that looks like one image is often multiple layers — a background watercolor/gradient, a foreground UI mockup PNG, an overlay icon. Inspect each container's full DOM tree and enumerate ALL `<img>` elements and background images within it, including absolutely-positioned overlays. Missing an overlay image makes the clone look empty even if the background is correct.

### 4. Foundation First

Nothing can be built until the foundation exists: global CSS with the target site's design tokens (colors, fonts, spacing), TypeScript types for the content structures, and global assets (fonts, favicons). This is sequential and non-negotiable. Everything after this can be parallel.

### 5. Extract How It Looks AND How It Behaves

A website is not a screenshot — it's a living thing. Elements move, change, appear, and disappear in response to scrolling, hovering, clicking, resizing, and time. If you only extract the static CSS of each element, your clone will look right in a screenshot but feel dead when someone actually uses it.

For every element, extract its **appearance** (exact computed CSS via `getComputedStyle()`) AND its **behavior** (what changes, what triggers the change, and how the transition happens). Not "it looks like 16px" — extract the actual computed value. Not "the nav changes on scroll" — document the exact trigger (scroll position, IntersectionObserver threshold, viewport intersection), the before and after states (both sets of CSS values), and the transition (duration, easing, CSS transition vs. JS-driven vs. CSS `animation-timeline`).

Examples of behaviors to watch for — these are illustrative, not exhaustive. The page may do things not on this list, and you must catch those too:
- A navbar that shrinks, changes background, or gains a shadow after scrolling past a threshold
- Elements that animate into view when they enter the viewport (fade-up, slide-in, stagger delays)
- Sections that snap into place on scroll (`scroll-snap-type`)
- Parallax layers that move at different rates than the scroll
- Hover states that animate (not just change — the transition duration and easing matter)
- Dropdowns, modals, accordions with enter/exit animations
- Scroll-driven progress indicators or opacity transitions
- Auto-playing carousels or cycling content
- Dark-to-light (or any theme) transitions between page sections
- **Tabbed/pill content that cycles** — buttons that switch visible card sets with transitions
- **Scroll-driven tab/accordion switching** — sidebars where the active item auto-changes as content scrolls past (IntersectionObserver, NOT click handlers)
- **Smooth scroll libraries** (Lenis, Locomotive Scroll) — check for `.lenis` class or scroll container wrappers

### 6. Identify the Interaction Model Before Building

This is the single most expensive mistake in cloning: building a click-based UI when the original is scroll-driven, or vice versa. Before writing any builder prompt for an interactive section, you must definitively answer: **Is this section driven by clicks, scrolls, hovers, time, or some combination?**

How to determine this:
1. **Don't click first.** Scroll through the section slowly and observe if things change on their own as you scroll.
2. If they do, it's scroll-driven. Extract the mechanism: `IntersectionObserver`, `scroll-snap`, `position: sticky`, `animation-timeline`, or JS scroll listeners.
3. If nothing changes on scroll, THEN click/hover to test for click/hover-driven interactivity.
4. Document the interaction model explicitly in the component spec: "INTERACTION MODEL: scroll-driven with IntersectionObserver" or "INTERACTION MODEL: click-to-switch with opacity transition."

A section with a sticky sidebar and scrolling content panels is fundamentally different from a tabbed interface where clicking switches content. Getting this wrong means a complete rewrite, not a CSS tweak.

### 7. Extract Every State, Not Just the Default

Many components have multiple visual states — a tab bar shows different cards per tab, a header looks different at scroll position 0 vs 100, a card has hover effects. You must extract ALL states, not just whatever is visible on page load.

For tabbed/stateful content:
- Click each tab/button via browser MCP
- Extract the content, images, and card data for EACH state
- Record which content belongs to which state
- Note the transition animation between states (opacity, slide, fade, etc.)

For scroll-dependent elements:
- Capture computed styles at scroll position 0 (initial state)
- Scroll past the trigger threshold and capture computed styles again (scrolled state)
- Diff the two to identify exactly which CSS properties change
- Record the transition CSS (duration, easing, properties)
- Record the exact trigger threshold (scroll position in px, or viewport intersection ratio)

### 8. Spec Files Are the Source of Truth

Every component gets a specification file in `docs/research/components/` BEFORE any builder is dispatched. This file is the contract between your extraction work and the builder agent. The builder receives the spec file contents inline in its prompt — the file also persists as an auditable artifact that the user (or you) can review if something looks wrong.

The spec file is not optional. It is not a nice-to-have. If you dispatch a builder without first writing a spec file, you are shipping incomplete instructions based on whatever you can remember from a browser MCP session, and the builder will guess to fill gaps.

### 9. Build Must Always Compile

Every builder agent must verify `npx tsc --noEmit` passes before finishing. After merging worktrees, you verify `bun run build` passes (vite build + nitro). A broken build is never acceptable, even temporarily.

## Phase 1: Reconnaissance

Navigate to the target URL with browser MCP.

### Screenshots
- Take **full-page screenshots** at desktop (1440px) and mobile (390px) viewports
- Save to `docs/design-references/` with descriptive names
- These are your master reference — builders will receive section-specific crops/screenshots later

### Global Extraction
Extract these from the page before doing anything else:

**Fonts** — Inspect `<link>` tags for Google Fonts or self-hosted fonts. Check computed `font-family` on key elements (headings, body, code, labels). Document every family, weight, and style actually used. Load them in this project: Google Fonts get entries in the `links` array of the route `head()` (root route `src/routes/__root.tsx`, or the page route) — e.g. `{ rel: "stylesheet", href: "https://fonts.googleapis.com/..." }`, with `{ rel: "preconnect", href: "https://fonts.gstatic.com", crossOrigin: "anonymous" }` for the gstatic preconnect; self-hosted fonts get downloaded to `public/fonts/` and declared with `@font-face` rules at the top of `src/styles.css`. Then update the `--font-sans` / `--font-mono` / `--font-heading` values in the `@theme inline` block of `src/styles.css` to the real families.

**Colors** — Extract the site's color palette from computed styles across the page. Merge the target's actual colors into `src/styles.css` in the `@theme inline` block and the `:root` and `.dark` blocks (Tailwind v4 uses oklch tokens). Map them to shadcn's token names (background, foreground, primary, muted, etc.) where they fit. Add custom properties for colors that don't map to shadcn tokens.

**Favicons & Meta** — Download favicons, apple-touch-icons, OG images, webmanifest to `public/seo/`. Reference the favicon via the `links` array of the route `head()` (e.g. `{ rel: "icon", href: "/seo/favicon.png", type: "image/png" }`), and set `<title>`, description, and OG tags via the `meta` array of the route `head()` (root route for site-wide defaults, page route for per-page values).

**Global UI patterns** — Identify any site-wide CSS or JS: custom scrollbar hiding, scroll-snap on the page container, global keyframe animations, backdrop filters, gradients used as overlays, **smooth scroll libraries** (Lenis, Locomotive Scroll — check for `.lenis`, `.locomotive-scroll`, or custom scroll container classes). Add these to `src/styles.css` and note any libraries that need to be installed.

### Mandatory Interaction Sweep

This is a dedicated pass AFTER screenshots and BEFORE anything else. Its purpose is to discover every behavior on the page — many of which are invisible in a static screenshot.

**Scroll sweep:** Scroll the page slowly from top to bottom via browser MCP. At each section, pause and observe:
- Does the header change appearance? Record the scroll position where it triggers.
- Do elements animate into view? Record which ones and the animation type.
- Does a sidebar or tab indicator auto-switch as you scroll? Record the mechanism.
- Are there scroll-snap points? Record which containers.
- Is there a smooth scroll library active? Check for non-native scroll behavior.

**Click sweep:** Click every element that looks interactive:
- Every button, tab, pill, link, card
- Record what happens: does content change? Does a modal open? Does a dropdown appear?
- For tabs/pills: click EACH ONE and record the content that appears for each state

**Hover sweep:** Hover over every element that might have hover states:
- Buttons, cards, links, images, nav items
- Record what changes: color, scale, shadow, underline, opacity

**Responsive sweep:** Test at 3 viewport widths via browser MCP:
- Desktop: 1440px
- Tablet: 768px
- Mobile: 390px
- At each width, note which sections change layout (column → stack, sidebar disappears, etc.) and at approximately which breakpoint the change occurs.

Save all findings to `docs/research/BEHAVIORS.md`. This is your behavior bible — reference it when writing every component spec.

### Page Topology
Map out every distinct section of the page from top to bottom. Give each a working name. Document:
- Their visual order
- Which are fixed/sticky overlays vs. flow content
- The overall page layout (scroll container, column structure, z-index layers)
- Dependencies between sections (e.g., a floating nav that overlays everything)
- **The interaction model** of each section (static, click-driven, scroll-driven, time-driven)

**Selector robustness:** page builders name section containers differently — Elementor's modern flex containers are `.e-con.e-parent` (NOT `.elementor-section`), other builders use `section`, `[data-block]`, or plain `div` stacks. If your first selector returns fewer than a handful of sections on a long page, it's the selector that's wrong, not the page. Fall back to walking the DOM for direct children of the main content wrapper taller than ~100px.

**Completeness gate (mandatory):** sum the heights of the sections you mapped and compare against `document.body.scrollHeight`. If the sum accounts for less than ~90% of the page height, you have missed sections — fix the topology before building anything. A clone built from an incomplete map ships with entire sections silently missing, and nobody notices until the user does. Also scroll the full page first: lazy-loaded sections and images don't exist in the DOM until scrolled into view (read `data-src`/`data-lazy-src` for image URLs that haven't materialized).

Save this as `docs/research/PAGE_TOPOLOGY.md` — it becomes your assembly blueprint. **Build every section in the map. There is no "main sections" shortcut** — a clone of 4 sections out of 9 is not a smaller clone, it's a broken page.

## Phase 2: Foundation Build

This is sequential. Do it yourself (not delegated to an agent) since it touches many files:

1. **Load fonts** — add the target site's actual fonts via the `links` array of the route `head()` (Google Fonts) or `@font-face` rules in `src/styles.css` pointing at `public/fonts/` (self-hosted), then set the font token values in `src/styles.css`
2. **Update `src/styles.css`** with the target's color tokens (in `@theme inline` + `:root`/`.dark` oklch blocks), spacing values, keyframe animations, utility classes, and any **global scroll behaviors** (Lenis, smooth scroll CSS, scroll-snap on body). Set the document `<title>`, description, favicon, and OG tags via the root route `head()` in `src/routes/__root.tsx`
3. **Create TypeScript interfaces** in `src/types/` for the content structures you've observed
4. **Extract SVG icons** — find all inline `<svg>` elements on the page, deduplicate them, and save as named React components in `src/components/icons.tsx`. Name them by visual function (e.g., `SearchIcon`, `ArrowRightIcon`, `LogoIcon`).
5. **Download global assets** — write and run a Node.js script (`scripts/download-assets.mjs`) that downloads all images, videos, and other binary assets from the page to `public/`. Preserve meaningful directory structure.
6. Verify: `bun run build` passes

### Asset Discovery Script Pattern

Use browser MCP to enumerate all assets on the page:

```javascript
// Run this via browser MCP to discover all assets
JSON.stringify({
  images: [...document.querySelectorAll('img')].map(img => ({
    src: img.src || img.currentSrc,
    alt: img.alt,
    width: img.naturalWidth,
    height: img.naturalHeight,
    // Include parent info to detect layered compositions
    parentClasses: img.parentElement?.className,
    siblings: img.parentElement ? [...img.parentElement.querySelectorAll('img')].length : 0,
    position: getComputedStyle(img).position,
    zIndex: getComputedStyle(img).zIndex
  })),
  videos: [...document.querySelectorAll('video')].map(v => ({
    src: v.src || v.querySelector('source')?.src,
    poster: v.poster,
    autoplay: v.autoplay,
    loop: v.loop,
    muted: v.muted
  })),
  backgroundImages: [...document.querySelectorAll('*')].filter(el => {
    const bg = getComputedStyle(el).backgroundImage;
    return bg && bg !== 'none';
  }).map(el => ({
    url: getComputedStyle(el).backgroundImage,
    element: el.tagName + '.' + el.className?.split(' ')[0]
  })),
  svgCount: document.querySelectorAll('svg').length,
  fonts: [...new Set([...document.querySelectorAll('*')].slice(0, 200).map(el => getComputedStyle(el).fontFamily))],
  favicons: [...document.querySelectorAll('link[rel*="icon"]')].map(l => ({ href: l.href, sizes: l.sizes?.toString() }))
});
```

Then write a download script that fetches everything to `public/`. Use batched parallel downloads (4 at a time) with proper error handling.

## Phase 3: Component Specification & Dispatch

This is the core loop. For each section in your page topology (top to bottom), you do THREE things: **extract**, **write the spec file**, then **dispatch builders**.

### Step 1: Extract

For each section, use browser MCP to extract everything:

1. **Screenshot** the section in isolation (scroll to it, screenshot the viewport). Save to `docs/design-references/`.

2. **Extract CSS** for every element in the section. Use the extraction script below — don't hand-measure individual properties. Run it once per component container and capture the full output:

```javascript
// Per-component extraction — run via browser MCP
// Replace SELECTOR with the actual CSS selector for the component
(function(selector) {
  const el = document.querySelector(selector);
  if (!el) return JSON.stringify({ error: 'Element not found: ' + selector });
  const props = [
    'fontSize','fontWeight','fontFamily','lineHeight','letterSpacing','color',
    'textTransform','textDecoration','backgroundColor','background',
    'padding','paddingTop','paddingRight','paddingBottom','paddingLeft',
    'margin','marginTop','marginRight','marginBottom','marginLeft',
    'width','height','maxWidth','minWidth','maxHeight','minHeight',
    'display','flexDirection','justifyContent','alignItems','gap',
    'gridTemplateColumns','gridTemplateRows',
    'borderRadius','border','borderTop','borderBottom','borderLeft','borderRight',
    'boxShadow','overflow','overflowX','overflowY',
    'position','top','right','bottom','left','zIndex',
    'opacity','transform','transition','cursor',
    'objectFit','objectPosition','mixBlendMode','filter','backdropFilter',
    'whiteSpace','textOverflow','WebkitLineClamp'
  ];
  function extractStyles(element) {
    const cs = getComputedStyle(element);
    const styles = {};
    props.forEach(p => { const v = cs[p]; if (v && v !== 'none' && v !== 'normal' && v !== 'auto' && v !== '0px' && v !== 'rgba(0, 0, 0, 0)') styles[p] = v; });
    return styles;
  }
  function walk(element, depth) {
    if (depth > 4) return null;
    const children = [...element.children];
    return {
      tag: element.tagName.toLowerCase(),
      classes: element.className?.toString().split(' ').slice(0, 5).join(' '),
      text: element.childNodes.length === 1 && element.childNodes[0].nodeType === 3 ? element.textContent.trim().slice(0, 200) : null,
      styles: extractStyles(element),
      images: element.tagName === 'IMG' ? { src: element.src, alt: element.alt, naturalWidth: element.naturalWidth, naturalHeight: element.naturalHeight } : null,
      childCount: children.length,
      children: children.slice(0, 20).map(c => walk(c, depth + 1)).filter(Boolean)
    };
  }
  return JSON.stringify(walk(el, 0), null, 2);
})('SELECTOR');
```

3. **Extract multi-state styles** — for any element with multiple states (scroll-triggered, hover, active tab), capture BOTH states:

```javascript
// State A: capture styles at current state (e.g., scroll position 0)
// Then trigger the state change (scroll, click, hover via browser MCP)
// State B: re-run the extraction script on the same element
// The diff between A and B IS the behavior specification
```

Record the diff explicitly: "Property X changes from VALUE_A to VALUE_B, triggered by TRIGGER, with transition: TRANSITION_CSS."

4. **Extract real content** — all text, alt attributes, aria labels, placeholder text. Use `element.textContent` for each text node. For tabbed/stateful content, **click each tab and extract content per state**.

5. **Identify assets** this section uses — which downloaded images/videos from `public/`, which icon components from `icons.tsx`. Check for **layered images** (multiple `<img>` or background-images stacked in the same container).

6. **Assess complexity** — how many distinct sub-components does this section contain? A distinct sub-component is an element with its own unique styling, structure, and behavior (e.g., a card, a nav item, a search panel).

### Step 2: Write the Component Spec File

For each section (or sub-component, if you're breaking it up), create a spec file in `docs/research/components/`. This is NOT optional — every builder must have a corresponding spec file.

**File path:** `docs/research/components/<component-name>.spec.md`

**Template:**

```markdown
# <ComponentName> Specification

## Overview
- **Target file:** `src/components/<ComponentName>.tsx`
- **Screenshot:** `docs/design-references/<screenshot-name>.png`
- **Interaction model:** <static | click-driven | scroll-driven | time-driven>

## DOM Structure
<Describe the element hierarchy — what contains what>

## Computed Styles (exact values from getComputedStyle)

### Container
- display: ...
- padding: ...
- maxWidth: ...
- (every relevant property with exact values)

### <Child element 1>
- fontSize: ...
- color: ...
- (every relevant property)

### <Child element N>
...

## States & Behaviors

### <Behavior name, e.g., "Scroll-triggered floating mode">
- **Trigger:** <exact mechanism — scroll position 50px, IntersectionObserver rootMargin "-30% 0px", click on .tab-button, hover>
- **State A (before):** maxWidth: 100vw, boxShadow: none, borderRadius: 0
- **State B (after):** maxWidth: 1200px, boxShadow: 0 4px 20px rgba(0,0,0,0.1), borderRadius: 16px
- **Transition:** transition: all 0.3s ease
- **Implementation approach:** <CSS transition + scroll listener | IntersectionObserver | CSS animation-timeline | etc.>

### Hover states
- **<Element>:** <property>: <before> → <after>, transition: <value>

## Per-State Content (if applicable)

### State: "Featured"
- Title: "..."
- Subtitle: "..."
- Cards: [{ title, description, image, link }, ...]

### State: "Productivity"
- Title: "..."
- Cards: [...]

## Assets
- Background image: `public/images/<file>.webp`
- Overlay image: `public/images/<file>.png`
- Icons used: <ArrowIcon>, <SearchIcon> from icons.tsx

## Text Content (verbatim)
<All text content, copy-pasted from the live site>

## Responsive Behavior
- **Desktop (1440px):** <layout description>
- **Tablet (768px):** <what changes — e.g., "maintains 2-column, gap reduces to 16px">
- **Mobile (390px):** <what changes — e.g., "stacks to single column, images full-width">
- **Breakpoint:** layout switches at ~<N>px
```

Fill every section. If a section doesn't apply (e.g., no states for a static footer), write "N/A" — but think twice before marking States & Behaviors as N/A. Even a footer might have hover states on links.

### Step 3: Dispatch Builders

Based on complexity, dispatch builder agent(s) in worktree(s):

**Simple section** (1-2 sub-components): One builder agent gets the entire section.

**Complex section** (3+ distinct sub-components): Break it up. One agent per sub-component, plus one agent for the section wrapper that imports them. Sub-component builders go first since the wrapper depends on them.

**What every builder agent receives:**
- The full contents of its component spec file (inline in the prompt — don't say "go read the spec file")
- Path to the section screenshot in `docs/design-references/`
- Which shared components to import (`icons.tsx`, `cn()`, shadcn primitives)
- The target file path (e.g., `src/components/HeroSection.tsx`)
- Instruction to verify with `npx tsc --noEmit` before finishing
- For responsive behavior: the specific breakpoint values and what changes
- The constraint that this is a **TanStack Start SSR app**: the component is server-rendered then hydrated, so no `window` / `document` / `localStorage` at module scope or during render (guard all browser APIs in `useEffect`); no `"use client"` directives (that's Next.js); use plain `<img>`; use `<Link>` from `@tanstack/react-router` for internal nav; and add `suppressHydrationWarning` (or a mounted flag) for any time-based or random UI

**Don't wait.** As soon as you've dispatched the builder(s) for one section, move to extracting the next section. Builders work in parallel in their worktrees while you continue extraction.

### Step 4: Merge

As builder agents complete their work:
- Merge their worktree branches into main
- You have full context on what each agent built, so resolve any conflicts intelligently
- After each merge, verify the build still passes: `bun run build`
- If a merge introduces type errors, fix them immediately

The extract → spec → dispatch → merge cycle continues until all sections are built.

## Phase 4: Page Assembly

After all sections are built and merged, assemble the page in `src/routes/index.tsx` (its `component`):

- Import all section components
- Implement the page-level layout from your topology doc (scroll containers, column structures, sticky positioning, z-index layering)
- Connect real content to component props
- Set the page's `head()` (title, description, OG) on the route's `createFileRoute("/")({ head: () => ({...}), component: Index })`
- Implement page-level behaviors: scroll snap, scroll-driven animations, dark-to-light transitions, intersection observers, smooth scroll (Lenis etc.) — all browser-API usage guarded in `useEffect`
- **Multi-page clones:** each additional cloned page gets its own file under `src/routes/`, with paths mirroring the original site's URL structure. File-based routing regenerates `routeTree.gen.ts` at build — no manual route registration
- Verify: `bun run build` passes clean

## Phase 5: Visual QA Diff

After assembly, do NOT declare the clone complete — and do NOT push. This phase is a **hard gate**, not a suggestion: a clone that skips it ships with wrong sizing, missing layers, and invented styling that the builder never noticed. Take side-by-side comparison screenshots:

1. Start the local dev server (`bun run dev`), open the clone in the browser via browser MCP (use the dev server's printed URL/port), and open the original site the same way
2. **Completeness first:** count the sections in the rendered clone against `PAGE_TOPOLOGY.md`, and compare total page height against the original (same viewport). A large height gap = missing sections. Fix completeness before polishing pixels.
3. Compare section by section, top to bottom, at desktop (1440px)
4. Compare again at mobile (390px)
5. For each discrepancy found:
   - Check the component spec file — was the value extracted correctly?
   - If the spec was wrong: re-extract from browser MCP, update the spec, fix the component
   - If the spec was right but the builder got it wrong: fix the component to match the spec
6. Test all interactive behaviors: scroll through the page, click every button/tab, hover over interactive elements
7. Verify smooth scroll feels right, header transitions work, tab switching works, animations play

Only after this visual QA pass is the clone complete. **The push to the Lovable repo (Phase 6) happens after this gate, never before it.**

## Phase 6: Lovable Handoff

**Lovable cannot import an existing external GitHub repository.** Its GitHub integration is one-directional — Lovable *creates* a repo it owns and syncs to it; there is no "import repo as project" feature. So the handoff is not "import this repo into Lovable." Instead, the clone must be built to **match Lovable's scaffold** and pushed into a repo that **Lovable itself created**.

The working handoff:

1. **The Lovable-owned repo already exists — the user created it before running this skill** (that's where the Lovable repo URL argument came from). Only the user can do this: in Lovable they create a project, then click **Connect to GitHub → Create Repository** (an OAuth step only they can perform). Lovable creates a repo it owns and does the first sync. Sync is a single branch (usually `main`) and is two-way after creation: pushes to that branch flow back into Lovable. Pre-Flight step 4 already cloned this repo and verified you can push to it.
2. **Build the clone INSIDE the cloned Lovable repo** (you cloned it in Pre-Flight step 4 — that clone is your build workspace). Because Lovable's scaffold is a specific structure (confirmed against the real repo in Pre-Flight step 5), do the clone work there — port sections into the home route + `src/components/`, merge tokens into the scaffold's global stylesheet, wire fonts/favicon/OG via the route `head()` — rather than scaffolding from a generic template (which drifts as Lovable updates).
3. **Do NOT wholesale-overwrite Lovable's repo with a foreign scaffold.** Preserve `.lovable/`, `vite.config.ts`, `src/router.tsx`, `src/server.ts`, `src/start.ts`, `src/routes/__root.tsx`, `src/lib/lovable-error-reporting.ts`, and `package.json`. Add your components/data/hooks/assets and edit the route files + `src/styles.css` only.
4. **Verify before pushing.** Before declaring the job done:
   - `bun run build` passes from a fresh state (vite build + nitro); `npx tsc --noEmit` is clean
   - Lovable's scaffold files (step 3 list) are intact and unmodified except for intended edits
   - Exactly one `package.json`, at the repo root — no workspaces, no second package.json, no foreign framework added
   - `bun run dev` starts the dev server and serves the clone
   - All assets referenced by the clone live in `public/` (nothing hotlinked from the original site except where intentional)
   - Commit everything and push to the synced branch (usually `main`) of Lovable's repo

Then tell the user their next step in plain language: the clone has been pushed to the branch Lovable syncs, so Lovable will pull it in — open the Lovable project to see it, and the site is now theirs to edit in Lovable and publish. If the user instead only wants a standalone site (not Lovable), the same clone works as a plain build — the Lovable-repo steps are simply skipped.

5. **Archive the run in this template repo.** After the Lovable push succeeds, commit the extraction artifacts here too — `docs/research/<hostname>/`, `docs/design-references/<hostname>/`, `scripts/<hostname>/` — with a message like `docs: extraction artifacts for <hostname>`, and push to this template repo's `main`. This preserves an auditable per-site record (specs, screenshots, topology) and leaves the template clean for the next clone. Never commit `node_modules/`, the cloned Lovable repo working directory, or build output.

## Pre-Dispatch Checklist

Before dispatching ANY builder agent, verify you can check every box. If you can't, go back and extract more.

- [ ] Spec file written to `docs/research/components/<name>.spec.md` with ALL sections filled
- [ ] Every CSS value in the spec is from `getComputedStyle()`, not estimated
- [ ] Interaction model is identified and documented (static / click / scroll / time)
- [ ] For stateful components: every state's content and styles are captured
- [ ] For scroll-driven components: trigger threshold, before/after styles, and transition are recorded
- [ ] For hover states: before/after values and transition timing are recorded
- [ ] All images in the section are identified (including overlays and layered compositions)
- [ ] Responsive behavior is documented for at least desktop and mobile
- [ ] Text content is verbatim from the site, not paraphrased
- [ ] The builder prompt is under ~150 lines of spec; if over, the section needs to be split

## What NOT to Do

These are lessons from previous failed clones — each one cost hours of rework:

- **Don't build click-based tabs when the original is scroll-driven (or vice versa).** Determine the interaction model FIRST by scrolling before clicking. This is the #1 most expensive mistake — it requires a complete rewrite, not a CSS fix.
- **Don't extract only the default state.** If there are tabs showing "Featured" on load, click Productivity, Creative, Lifestyle and extract each one's cards/content. If the header changes on scroll, capture styles at position 0 AND position 100+.
- **Don't miss overlay/layered images.** A background watercolor + foreground UI mockup = 2 images. Check every container's DOM tree for multiple `<img>` elements and positioned overlays.
- **Don't rebuild as styled text what the original ships as artwork.** Badges, pills, decorated headings, and underline flourishes are often exported PNGs/SVGs — that's why they have angled edges, gradients, or hand-drawn shapes CSS won't reproduce. Telltale sign: your DOM text search for copy you can see on screen returns nothing. When that happens, look for an `<img>` at that position and use the downloaded artwork instead of approximating it in CSS.
- **Don't hand-build UI that is actually a third-party embed.** Newsletter forms (Beehiiv, ConvertKit, Mailchimp), videos (Wistia, Vimeo, YouTube), calendars (Calendly), and chat widgets live in iframes. Enumerate every `<iframe>` on the page early — reusing the same public embed URL is both pixel-identical AND functional, while a hand-built imitation is neither. Also check for an invented overlay: if a section looks darker in your clone than the original, you probably added an overlay the original doesn't have — extract, don't assume.
- **Don't build mockup components for content that's actually videos/animations.** Check if a section uses `<video>`, Lottie, or canvas before building elaborate HTML mockups of what the video shows.
- **Don't approximate CSS classes.** "It looks like `text-lg`" is wrong if the computed value is `18px` and `text-lg` is `18px/28px` but the actual line-height is `24px`. Extract exact values.
- **Don't build everything in one monolithic commit.** The whole point of this pipeline is incremental progress with verified builds at each step.
- **Don't reference docs from builder prompts.** Each builder gets the CSS spec inline in its prompt — never "see DESIGN_TOKENS.md for colors." The builder should have zero need to read external docs.
- **Don't skip asset extraction.** Without real images, videos, and fonts, the clone will always look fake regardless of how perfect the CSS is.
- **Don't give a builder agent too much scope.** If you're writing a builder prompt and it's getting long because the section is complex, that's a signal to break it into smaller tasks.
- **Don't bundle unrelated sections into one agent.** A CTA section and a footer are different components with different designs — don't hand them both to one agent and hope for the best.
- **Don't skip responsive extraction.** If you only inspect at desktop width, the clone will break at tablet and mobile. Test at 1440, 768, and 390 during extraction.
- **Don't forget smooth scroll libraries.** Check for Lenis (`.lenis` class), Locomotive Scroll, or similar. Default browser scrolling feels noticeably different and the user will spot it immediately.
- **Don't dispatch builders without a spec file.** The spec file forces exhaustive extraction and creates an auditable artifact. Skipping it means the builder gets whatever you can fit in a prompt from memory.
- **Don't break Lovable compatibility.** SSR is REQUIRED — Lovable's scaffold is TanStack Start (SSR). The failure mode is the opposite: don't overwrite or ignore Lovable's scaffold, don't introduce a conflicting framework (e.g. Next.js) or a second `package.json`, and don't add `"use client"` directives or `react-router-dom`. Preserve `.lovable/`, `vite.config.ts`, `src/router.tsx`, `src/server.ts`, `src/start.ts`, `src/routes/__root.tsx`, and `src/lib/lovable-error-reporting.ts`. If you drift from Lovable's scaffold, the push won't line up with what Lovable syncs and the whole point of this pipeline is lost.

## Completion

When done, report:
- Total sections built
- Total components created
- Total spec files written (should match components)
- Total assets downloaded (images, videos, SVGs, fonts)
- Build status (`bun run build` result)
- Visual QA results (any remaining discrepancies)
- Lovable handoff status (Phase 6: scaffold preserved, `bun run build` clean, ported into the Lovable-created repo and pushed to the synced branch yes/no)
- Template archive status (extraction artifacts committed to this template repo under the site's hostname folders and pushed yes/no)
- Any known gaps or limitations
