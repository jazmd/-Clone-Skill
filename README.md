# Website Cloner — Lovable Edition

Rebuild any website you own into a Lovable project you can edit and publish. No terminal. No code. One command — run entirely from a browser tab.

Point Claude Code at a site, and it inspects the live page, extracts the real design values and assets, rebuilds every section, checks its own work against the original, and delivers the result straight into your Lovable project.

Built on the open-source [ai-website-cloner-template](https://github.com/JCodesMore/ai-website-cloner-template) by JCodesMore (MIT).

---

## How this works (read this first — it saves confusion)

There are **two** GitHub repos in play, and keeping them straight is the whole game:

1. **Your tool repo** — the copy you make of *this* template, **once, ever**. It holds the cloning skill and a headless browser. It's the repo you open in Claude Code, and you reuse it for every site you clone. **Nothing gets built here.**
2. **Your Lovable repos** — **one per site**, created by **Lovable** when you connect a Lovable project to GitHub. This is where each cloned website actually gets built and delivered.

> **Why not just "import my repo into Lovable"?** Because Lovable can't. Lovable's GitHub feature only works one way: Lovable *creates* a repo it owns and syncs to it — there's no "import an existing repo" button (we checked, it genuinely doesn't exist). So instead of importing, the skill builds your clone *inside the repo Lovable made* and pushes it there. Lovable sees the push and pulls it in. That's the trick that makes this work.

## What You Need

- A Claude account with Claude Code access (Pro or Max)
- A GitHub account (free)
- A Lovable account

## Step-by-Step (cloud — zero install)

The setup splits cleanly in two: a **one-time setup** you do once ever, and a short **per-site loop** you repeat for each website you clone.

### Part A: One-time setup (do this once, ever)

1. On **this** repo's GitHub page, click **Use this template → Create a new repository**. Name it anything you like — `website-cloner` is perfect. This copy is your **cloning tool**, and you'll reuse it for every site you ever clone. (You do NOT make a new copy per site — see the FAQ.)
2. Give Claude access to **All repositories** on GitHub. This matters more than it looks: every clone gets delivered into a *new* repo that Lovable creates later, and "All repositories" means Claude can push to those future repos without you ever touching GitHub settings again. Where you find this option depends on whether you've connected before:
   - **First time connecting?** Go to [claude.com/code](https://claude.com/code) and connect GitHub when asked. GitHub shows an install screen asking which repositories Claude can access — choose **All repositories** there.
   - **Already connected Claude Code to GitHub before?** That screen won't appear again (the option is *not* in Claude Code itself — it lives on GitHub). Go to [github.com/settings/installations](https://github.com/settings/installations), find the **Claude** app → **Configure** → under **Repository access** choose **All repositories** → **Save**.
3. Open your tool repo in Claude Code.
4. **Turn on network access.** In the session's environment settings, allow network access so Claude can reach the sites you're cloning and download its browser. *If a clone fails right at the start, this is almost always why.*

### Part B: For every site you clone (repeat this loop)

#### 1. Create a Lovable project and let Lovable make its repo

1. In **Lovable**, click **New Project**. Give it any quick prompt (like `blank landing page`) just so the project exists — you'll be overwriting it.
2. Open the **GitHub** menu in the Lovable editor (top-right, or **Settings → GitHub**) → **Connect to GitHub** → authorize → **Create Repository**.
3. Lovable creates a repo on your GitHub account. Click through to it and **copy its URL** — it looks like `https://github.com/yourname/your-project.git`. **You'll paste this into the command in the next step.**

#### 2. Run the clone

Back in your **same tool repo** in Claude Code, type the command with **two things**: the site you're cloning, then the **Lovable repo URL** you just copied:

```
/clone-website https://the-site-you-own.com https://github.com/yourname/your-project.git
```

That second URL is what tells Claude *where to deliver* your clone. If you forget it, Claude will stop and ask — it can't guess, because the repo it's sitting in (your tool) isn't your Lovable project.

Then let it work. A full clone of a real business site takes a while — you can close the tab and check back, or watch from the Claude mobile app.

#### 3. Open it in Lovable

When Claude reports the clone is built and pushed, open your **Lovable project**. Lovable pulls in the pushed code and your cloned site appears in the editor — yours to tweak visually and **Publish**.

Cloning another site? Go back to the top of Part B — new Lovable project, same tool repo.

## Local / Desktop (optional)

The same `/clone-website` command works in a Claude Code session on your own machine (Claude desktop app, Code tab, or the CLI). The bundled headless browser in `.mcp.json` works there too. Only needed if you prefer working locally.

---

## FAQ / Troubleshooting

### "Do I make a new copy of the template for each site I clone?"

**No — one tool repo, forever.** The tool repo is the workshop, not the product: nothing gets built in it, so there's nothing site-specific about it. What you create per site is a new **Lovable project** (Part B, step 1) — that's not this tool being fussy, it's how Lovable works: each Lovable project is one website, with its own editor, its own Publish button, and its own GitHub repo. So cloning 50 sites = 1 tool repo + 50 Lovable projects. Your tool repo keeps research notes from each clone in its own folder (`docs/research/<site>/`), so repeated clones never collide.

### ⭐ "Claude says it can't push to / can't access my Lovable repo"

**This is the most common snag, and it's a GitHub permission thing — not a mistake you made.** It means Claude Code's GitHub access was limited to selected repositories and your new Lovable repo isn't on the list — this is exactly what choosing **All repositories** during one-time setup (Part A, step 2) prevents. Claude can read the Lovable repo but can't deliver your clone into it.

The skill checks for this **up front** (before spending an hour cloning) and will stop and tell you. Here's the one-time fix:

1. On GitHub, click your avatar (top-right) → **Settings**.
2. Left sidebar → **Applications** (under "Integrations") → **Claude Code** (the GitHub App) → **Configure**.
3. Under **Repository access**, either add your Lovable repo to the selected list, or choose **All repositories**.
4. Click **Save**, go back to Claude Code, and tell it to **retry**.

*(This grants Claude permission to push your finished clone into your Lovable repo. It's the single most important step to get right.)*

### "It failed at the very first step / can't reach the site"

Network access isn't enabled in your cloud session. Open the session's environment settings, turn on network access, and retry. This lets Claude reach the site and download its browser.

### "Where do I find my Lovable repo URL?"

Inside your Lovable project, open the **GitHub** menu — it links to the repo Lovable created. On that GitHub page, use the green **Code** button (or the address bar) to copy the `https://github.com/...git` URL. That's the second argument in the command.

### "Do I need to know how to code?"

No. Everything happens in the browser: Lovable, GitHub, and Claude Code on the web. You paste two links into one command.

### "How long does a clone take?"

A simple page is quick; a full multi-section business site can take a while (lots of careful extraction and building). You can close the tab and come back — Claude keeps working, and the mobile app shows progress.

### "Will the countdown / live bits work?"

Static and animated sections are rebuilt faithfully. Anything genuinely dynamic (a live registration count, real payments) is emulated for looks — wire up real data later in Lovable.

## Use This For

- Rebuilding a site you own from WordPress, Wix, Webflow, or Squarespace into a Lovable project you control
- Recovering a site whose source code is lost — the developer left, the repo is gone, the platform is legacy
- Migrating a client's site (with their permission) into Lovable so they can finally edit it themselves

## Not For

- Cloning websites you don't own or have permission to rebuild
- Phishing, impersonation, or passing off someone else's design, brand, or copy as your own
- Violating a site's terms of service — some sites prohibit scraping or reproduction; check first

## A note on Lovable's stack

Lovable's underlying stack changes over time (as of August 2026 it's TanStack Start + SSR + React 19 + Tailwind v4). The skill doesn't hardcode this — it reads whatever Lovable actually generates in your repo and adapts. So this keeps working even after Lovable updates.

## Credits & License

MIT. Original template by [JCodesMore](https://github.com/JCodesMore/ai-website-cloner-template). Lovable-retarget, cloud configuration, and skill rewrite by Brian Hanson. See `LICENSE`.
