# Website Cloner — Lovable Edition

## What This Is

A **skill-delivery template** for reverse-engineering any website into a Lovable project, using Claude Code. This repo ships three things and nothing else:

- The `/clone-website` skill — `.claude/skills/clone-website/SKILL.md` (the source of truth for the workflow)
- A headless Playwright browser MCP, preconfigured in `.mcp.json` (runs via `npx`, nothing to install)
- The `docs/` structure the skill writes its extraction artifacts into

**There is no app to build in this repo.** The clone is built inside a *separate* clone of the user's **Lovable repo**, which the user creates in Lovable. This template is only the starting point for a Claude Code session — don't try to run, preview, or build it.

## The Build Target: Lovable's Live Scaffold

The clone is built to match **whatever Lovable currently scaffolds**. As of **August 2026** that is:

- **Framework:** TanStack Start (`@tanstack/react-start` + `@tanstack/react-router`, file-based routing), SSR via **nitro**, React **19**, TypeScript
- **Styling:** Tailwind **v4** + shadcn/ui "new-york". The Tailwind entry is **`src/styles.css`** (`@import "tailwindcss"` + `@theme inline` + oklch tokens). No `tailwind.config` / `postcss.config`.
- **No `index.html`.** The document shell is `src/routes/__root.tsx`; head/meta/fonts/favicon are set via a route's `head()` export.
- **Pages = routes** under `src/routes/`; the home page is `src/routes/index.tsx`. No `src/pages`, no `react-router-dom`.
- **Package manager:** bun. Build/verify with `bun run build` (vite + nitro).

**Do not treat these facts as permanent.** Lovable has changed its stack without notice before (it was a Vite SPA before this) and will again. The skill's job is to read the *actual* freshly-cloned Lovable repo and confirm the shape from what's there — adapting if it differs, not forcing this structure.

## How a Run Works

1. **User creates the Lovable-owned repo** — a Lovable project → **Connect to GitHub → Create Repository**. Only the user can do this (OAuth). Lovable owns and syncs the repo (one branch, usually `main`, two-way).
2. **User copies this template**, opens it in Claude Code (web or desktop), and runs:
   `/clone-website <target-url> <lovable-repo-url>`
3. **The skill** pre-flights (network, target reachable, repo **clonable + pushable**), clones the Lovable repo as its build workspace, extracts the target section-by-section, ports each section into the Lovable scaffold, verifies `bun run build`, and pushes to the synced branch. Lovable pulls it in. Finally it commits the run's extraction artifacts back to this template repo.

**This template is reused across every clone the user runs** — one tool repo, many sites over time. All extraction artifacts are therefore isolated per site in hostname folders (`docs/research/<hostname>/`, `docs/design-references/<hostname>/`, `scripts/<hostname>/`) and committed to this repo after each successful run, so runs never collide and each clone leaves an auditable record.

## Hard Constraints (Lovable compatibility)

- **Build inside the cloned Lovable repo** — never turn this template into the deliverable, and never scaffold a fresh framework from a generic template (it drifts from Lovable).
- **Never clobber Lovable's scaffold.** Preserve `.lovable/`, `vite.config.ts`, `src/router.tsx`, `src/server.ts`, `src/start.ts`, `src/routes/__root.tsx`, `src/lib/lovable-error-reporting.ts`, and `package.json`. Merge design tokens into the scaffold's `src/styles.css`; add routes/components/assets; edit the route files — don't replace config.
- **Exactly one `package.json`** in the Lovable repo — no second package.json, no workspaces, no foreign framework (Next.js, etc.), no `"use client"`.
- **SSR gotchas:** no bare `window` / `document` / `localStorage` during render (guard in `useEffect`); add `suppressHydrationWarning` for time-/random-based UI (countdowns, "spots left").
- **Only push after `bun run build` passes.** Pushing to the synced branch (usually `main`) is the delivery mechanism — a broken build must never be pushed.

## Per-Site Requirement Overrides

If `docs/research/<hostname>/SEO_REQUIREMENTS.md` exists, **read it before starting the
run and treat it as binding.** It overrides the skill's defaults — including the skill's
default of cloning only the single page the target URL resolves to, and its listing of
"SEO optimization" as out of scope. A site being migrated off a live, ranking domain has
requirements the generic workflow does not cover, and silently applying the defaults to
such a site destroys the client's search traffic.

Currently present: `docs/research/iloveahandyman.com/SEO_REQUIREMENTS.md` (full-site
clone + SEO/AEO parity required).

## Design Principles (for the clone)

- **Pixel-perfect emulation** — match the target's spacing, colors, typography exactly, from `getComputedStyle()` values, not estimates
- **Real content** — use actual text and assets from the target site, not placeholders
- **Match 1:1 first** — no personal aesthetic changes during the emulation phase

## This Template's Files

```
.claude/skills/clone-website/SKILL.md   # the skill — read this first
.mcp.json                               # headless Playwright MCP (npx, no install)
docs/
  research/
    INSPECTION_GUIDE.md                 # what to capture from a target site
    <hostname>/                         # per-site extraction output (tokens, components, layout)
  design-references/
    <hostname>/                         # per-site screenshots and visual references
scripts/
  <hostname>/                           # per-site asset-download / capture scripts
README.md                               # student-facing walkthrough
```

## Code Style (for ported code, inside the Lovable repo)

- TypeScript strict, no `any`; PascalCase components, camelCase utils
- Tailwind utility classes, no inline styles; 2-space indentation

## MOST IMPORTANT NOTES

- When launching Claude Code agent teams for the build, have each teammate work in its own worktree branch and merge at the end — you serve the orchestrator role with full context.
- The finished clone must pass the Phase 6 Lovable Handoff checklist in the skill (scaffold preserved, `bun run build` clean, pushed to the synced branch) before the job is declared done.

@docs/research/INSPECTION_GUIDE.md
