# Montis Newsletter — Complete System Guide

**Automated newsletter pipeline: Make.com + GitHub + Cloudflare + Claude MCP**

Last updated: June 2026 — Author: Matteo Marochini

> **Who this guide is for:** Anyone taking over this project — technical or not. By the end, you should be able to explain how the system works, fix it when it breaks, extend it safely, and even rebuild it from zero if needed.

---

## TL;DR (read this first)

Every morning, a robot (Make.com) wakes up, reads news from several finance/crypto websites (RSS feeds), asks an AI to summarize them, builds a webpage (HTML) out of those summaries, and saves that page to GitHub. A second robot (Cloudflare) serves that page to anyone who visits the newsletter's website. Nobody has to do anything by hand — except occasionally update the *look* of the page (the template) or approve a new news source.

Two automations exist:
- **Daily Digest** — runs Monday–Friday at 07:30 Paris time
- **Weekly Digest** — runs every Monday at 07:30 Paris time, plus includes live market prices

---

## Table of Contents

1. [Glossary — plain-language definitions](#1-glossary--plain-language-definitions)
2. [Big picture architecture](#2-big-picture-architecture)
3. [The Make.com scenarios in detail](#3-the-makecom-scenarios-in-detail)
4. [RSS feeds: what they are and how to manage them](#4-rss-feeds-what-they-are-and-how-to-manage-them)
5. [GitHub: where the content lives](#5-github-where-the-content-lives)
6. [Cloudflare Workers: how the site is actually served](#6-cloudflare-workers-how-the-site-is-actually-served)
7. [Using Claude + Make MCP to manage everything](#7-using-claude--make-mcp-to-manage-everything)
8. [Visual Studio Code: local editing workflow](#8-visual-studio-code-local-editing-workflow)
9. [Rebuilding this system from scratch (step-by-step)](#9-rebuilding-this-system-from-scratch-step-by-step)
10. [Golden rules / best practices](#10-golden-rules--best-practices)
11. [Troubleshooting](#11-troubleshooting)
12. [Handover checklist](#12-handover-checklist)

---

## 1. Glossary — plain-language definitions

If you're not technical, read this section first. Everything else will make more sense.

| Term | What it actually means |
|------|------------------------|
| **RSS feed** | A simple, machine-readable list of a website's latest articles (title, summary, link, date). Think of it as a website's "table of contents," but for robots to read. |
| **Make.com (Make)** | A no-code automation tool. You build a flow of "modules" (steps) that run one after another, like a flowchart. No programming required — just connecting boxes. |
| **Scenario** | One automated flow inside Make. We have two: "Daily Digest" and "Weekly Digest." |
| **Module** | One single step inside a scenario (e.g., "fetch this RSS feed," "ask the AI to summarize," "save to GitHub"). |
| **LLM / AI model** | The artificial intelligence that reads articles and writes short summaries. We use Llama 3.3 through a service called OpenRouter. |
| **OpenRouter** | A middleman service that lets Make "talk" to different AI models (like Llama) through one simple connection. |
| **HTML** | The code behind every webpage. It's just text with tags that tell a browser "this is a title," "this is a paragraph," etc. |
| **Template** | An HTML file with blank spots (placeholders) like `DATE_PLACEHOLDER`. Make fills in those blanks every day with fresh content. |
| **GitHub** | An online storage service for code and files, with version history (like "Track Changes" for code). Our newsletter's HTML files live here. |
| **Repository (repo)** | A "project folder" on GitHub. Ours is called `MMaro06/montis-newsletter`. |
| **Commit** | A saved snapshot of a change, with a short message explaining what changed. |
| **API** | A way for two computer systems to talk to each other automatically (instead of a human clicking buttons). |
| **Cloudflare Workers** | A service that runs small bits of code on the internet to serve our webpage to visitors, very fast, anywhere in the world. |
| **MCP (Model Context Protocol)** | A way to let an AI assistant (like Claude) directly operate other tools — for example, letting Claude edit a Make scenario for you, instead of you clicking through Make's interface yourself. |
| **Placeholder** | A marker word inside the HTML template (e.g. `REG_PLACEHOLDER`) that Make replaces with real content every run. |
| **SHA (in GitHub context)** | A unique fingerprint identifying the *current* version of a file. GitHub requires you to provide it when updating a file, to make sure you're not overwriting someone else's more recent change. |

---

## 2. Big picture architecture

Here's the whole system in one diagram:

```
 ┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐     ┌────────────┐
 │  RSS Feeds  │ --> │   Make.com   │ --> │  OpenRouter │ --> │  Make.com │ --> │   GitHub   │
 │ (news sites)│     │ (fetch news) │     │ (AI summary)│     │ (build    │     │ (save HTML)│
 └─────────────┘     └──────────────┘     └─────────────┘     │  HTML)    │     └─────┬──────┘
                                                                └──────────┘           │
                                                                                        v
                                                                              ┌───────────────────┐
                                                                              │ Cloudflare Workers │
                                                                              │  (serves the page  │
                                                                              │   to visitors)     │
                                                                              └───────────────────┘
```

### The 5 building blocks

| Component | Role | Specifics |
|-----------|------|-----------|
| **Make.com** (eu1.make.com) | The "robot" that runs the whole pipeline on a schedule | 2 active scenarios |
| **RSS feeds** | The raw news sources | 8+ sources (Ledger Insights, PostTrade360, etc.) |
| **OpenRouter** | The AI summarizer | Model: `meta-llama/llama-3.3-70b-instruct`, ~$0.002/day |
| **GitHub** | Where the final HTML pages and templates are stored | Repo: `MMaro06/montis-newsletter`, branch: `main` |
| **Cloudflare Workers** | The web server that shows the page to the public | `montis-newsletter.letzgomontis.workers.dev` |

### What happens, step by step, every morning at 07:30 Paris time

1. Make's internal clock triggers the scenario.
2. Make fetches the HTML **template** from GitHub (the page layout with blank placeholders).
3. Make fetches fresh articles from each **RSS feed**.
4. Make sends those articles to **OpenRouter**, which asks the AI to summarize them.
5. Make plugs each AI summary into the right **placeholder** in the template (e.g., `DLT_PLACEHOLDER` becomes a real paragraph about DLT news).
6. Make asks GitHub "what's the current fingerprint (SHA) of this file?" — required before saving.
7. Make pushes (saves) the finished HTML file to GitHub.
8. Cloudflare Workers automatically serves that updated file to anyone visiting the newsletter website.

No human action needed for any of this — it just runs.

---

## 3. The Make.com scenarios in detail

### 3.1 Daily Digest (Scenario ID: 6200212)

| Property | Value |
|----------|-------|
| Schedule | Monday–Friday, 07:30 Paris time |
| RSS sources | Ledger Insights, PostTrade360, The Block, Future of Finance, Cryptoast, crypto.news (Digital Assets tag), France Post-Marché, CoinTelegraph |
| Template placeholders (7) | `DATE_PLACEHOLDER`, `TLDR_PLACEHOLDER`, `MONTIS_PLACEHOLDER`, `DLT_PLACEHOLDER`, `REG_PLACEHOLDER`, `SUM_PLACEHOLDER`, `GEO_PLACEHOLDER` |
| Output file | `daily-digest.html` (pushed to GitHub `main` branch) |

### 3.2 Weekly Digest (Scenario ID: 6220310, internally named "Weekly Digest — Lundi 7h30")

| Property | Value |
|----------|-------|
| Schedule | Every Monday, 07:30 Paris time |
| Module count | 13 modules total |
| Module breakdown | 1 HTTP GET (template) + 2 RSS fetches + 6 AI/LLM calls + 1 SetVariable for the week's date + 1 SetVariable for the final HTML (chains 7 `replace()` calls) + 1 GitHub GET (fetch current SHA) + 1 GitHub PUT (save new content) |
| Template placeholders (7) | `WEEK_PLACEHOLDER` (used 3 times), `BIG_PLACEHOLDER`, `COMP_PLACEHOLDER`, `REG_PLACEHOLDER`, `TOK_PLACEHOLDER`, `MOV_PLACEHOLDER`, `READ_PLACEHOLDER` |
| RSS sources | Ledger Insights, PostTrade360 |
| Live market data | CoinGecko API (crypto prices) + Yahoo Finance via the `allorigins.win` CORS proxy (stock prices) — both run as client-side JavaScript directly in the visitor's browser, *not* through Make |
| Confirmed run | June 17, 2026 — status: success, 13 operations used, ~96 seconds total runtime |

### 3.3 Setup/Debug scenario (Scenario ID: 6199804)

This is a reusable "scratchpad" scenario used for one-off testing or maintenance tasks. **It must stay deactivated** when not actively being used — never leave it running.

---

## 4. RSS feeds: what they are and how to manage them

### What's inside an RSS feed?

It's a simple XML file. For each article, it lists:
- Title
- Short description
- Link to the full article
- Publish date

Make reads this list automatically every time the scenario runs.

### Current sources (Daily Digest)

| Source | Covers | Status |
|--------|--------|--------|
| Ledger Insights | DLT & Settlement | ✅ Active |
| PostTrade360 | Post-Trade | ✅ Active |
| The Block | Crypto & DLT | ✅ Active |
| Future of Finance | Fintech & Banking | ✅ Active |
| Cryptoast | Crypto (EU) | ✅ Active |
| crypto.news | Digital Assets | ✅ Active |
| France Post-Marché | Post-Trade (FR) | ✅ Active |
| CoinTelegraph | Crypto & News | ✅ Active |

### Current sources (Weekly Digest)

| Source | Covers | Status |
|--------|--------|--------|
| Ledger Insights | DLT & Settlement | ✅ Active |
| PostTrade360 | Post-Trade | ✅ Active |

### Candidates being considered

- **BIS (Bank for International Settlements)** press releases — likely has a working RSS feed, not yet confirmed/added
- **ESMA** (European Securities and Markets Authority) — investigated, but their site runs on Drupal CMS with no exposed public RSS URL found so far

### How to safely add a new RSS source

⚠️ **Golden rule: never touch Make until you've personally confirmed the feed URL actually works.**

Some sites *advertise* an RSS feed in their UI but actually gate it behind a login. In Make, this kind of failure can be silent (no visible error) if `stopOnHttpError` is set to `false` — which means broken sources can sit unnoticed for a long time.

**Step-by-step:**

1. Find the candidate feed — look for a link ending in `/rss`, `/feed`, `.xml`, or an RSS icon on the site.
2. **Test it yourself first**, outside of Make:
   ```
   curl -s "https://example.com/feed" | head -50
   ```
   You should see readable XML with article titles inside `<item>` tags. If you get an HTML login page, a 403/404 error, or garbage — the feed doesn't work. Stop here.
3. Once confirmed working, **share the URL for approval** before changing anything (Matteo or whoever owns the project prefers to approve new sources before they go live).
4. After approval, in Make:
   - Add an **HTTP > Get a file** (or "Make a request") module pointing to the feed URL, with "Parse response" turned ON.
   - Add an **OpenRouter (HTTP request)** module right after it to summarize the new articles.
   - Plug that summary into the correct placeholder inside the **SetVariable2** module that builds the final HTML.
5. Run the scenario once manually ("Run once" button in Make) to confirm it works before letting it run on schedule.

---

## 5. GitHub: where the content lives

### Repository details

- **Repo:** `MMaro06/montis-newsletter`
- **Branch used:** `main` (the only branch — there's no separate dev/staging branch)
- **Connection used by Make:** GitHub connection ID `8246786` (personal account `MMaro06`)

### Key files

| File | Purpose | Who edits it |
|------|---------|---------------|
| `daily-digest.html` | The finished Daily newsletter page | Auto-generated by Make — never edit by hand |
| `weekly-digest.html` | The finished Weekly newsletter page | Auto-generated by Make — never edit by hand |
| `daily-digest-template.html` | The blank layout with placeholders for Daily | Edited manually, uploaded via GitHub |
| `weekly-digest-template.html` | The blank layout with placeholders for Weekly | Edited manually, uploaded via GitHub |
| `README.md` | Project documentation | Edited manually |

**Rule of thumb:** structural/visual changes (colors, layout, fonts) go through manual edits to the *template* files on GitHub. Day-to-day content changes happen automatically through Make — you never touch those.

### How Make actually talks to GitHub

Two API calls happen at the end of every scenario run:

1. **GitHub GET** — fetches the current file's metadata. The important part of the response is the file's fingerprint, found at:
   ```
   2.body.sha
   ```
   (Not `2.sha`, not `2.data.sha` — this exact path matters and is a common mistake.)

2. **GitHub PUT** — pushes the new content. This call needs:
   - The new HTML content (base64-encoded)
   - The SHA from step 1 (proves you're updating the latest version, not overwriting someone else's change)
   - A commit message (auto-generated, but must not be empty)

If the SHA is missing or wrong, GitHub responds with a `422 Unprocessable Entity` error.

---

## 6. Cloudflare Workers: how the site is actually served

- **Public URL:** `montis-newsletter.letzgomontis.workers.dev`
- **What it does:** fetches the HTML files straight from GitHub (`raw.githubusercontent.com`) and serves them to anyone who visits.
- **Live market data:** For the Weekly Digest's "Markets Snapshot" section, prices come from CoinGecko and Yahoo Finance (via the `allorigins.win` proxy to bypass browser security restrictions). This happens live, in the visitor's own browser, using JavaScript — it is **not** generated by Make and is **not** stored on GitHub. Every visitor sees live, up-to-the-minute prices.

---

## 7. Using Claude + Make MCP to manage everything

### What is MCP, really?

**MCP (Model Context Protocol)** lets an AI assistant like Claude directly control other tools on your behalf — instead of you manually clicking through the Make.com interface, you can just describe what you want in plain language, and Claude does the clicking (so to speak) for you.

**Example:** Instead of logging into Make, finding the right scenario, adding modules one by one — you simply tell Claude:

> "Add the BIS press release feed to the Daily Digest scenario."

And Claude does it directly through the Make connector.

### How to turn it on

1. Go to claude.ai
2. Open **Settings → Connectors**
3. Search for "Make"
4. Click **Connect**, then log in with your Make.com account when prompted

### Example: adding a new RSS source through Claude

**Your prompt:**
> "I need to add the BIS press releases feed (https://www.bis.org/press/rss.xml) to the Daily Digest scenario (ID 6200212). Add an HTTP module to fetch it, an OpenRouter module to summarize the articles, and plug the summary into the HTML."

**What Claude will do:**
1. Open the scenario and review its current structure
2. Add the new modules in the right place
3. Test the run and report back what happened

### Things Claude (or you) must never do via MCP

- ❌ Never activate or modify the **Setup/Debug scenario (6199804)** unless intentionally using it as a one-off test runner — and deactivate it again afterward.
- ❌ Never try to upload large HTML templates as base64 text *inside* a Make module — see the size-limit warning in section 10.
- ❌ Never skip the "test it manually first" step from section 4 just because Claude can move fast — speed isn't worth a silent broken feed.

---

## 8. Visual Studio Code: local editing workflow

### Why use VS Code at all?

Make doesn't have a proper code editor — editing big HTML files inside its boxes is painful and error-prone. VS Code lets you:

- Edit the HTML templates comfortably, with syntax highlighting
- Preview your changes before pushing them live
- Use Git (version control) properly
- Catch mistakes before they reach the live website

### One-time setup: clone the repository

1. Open VS Code
2. Press `Ctrl+Shift+P` (Mac: `Cmd+Shift+P`) to open the command palette
3. Type and select **"Git: Clone"**
4. Paste: `https://github.com/MMaro06/montis-newsletter`
5. Choose a folder on your computer to store it
6. Click **"Open"** when prompted

### Day-to-day workflow: editing and publishing a template change

1. Open `daily-digest-template.html` or `weekly-digest-template.html` in the editor
2. Make your change (e.g., adjust a color, fix a typo, add a section)
3. **Preview it** before pushing — right-click the file → "Open with Live Server" (see extension below), or simply double-click the file to open it in your browser
4. Open the Source Control tab (`Ctrl+Shift+G`)
5. Write a clear commit message (e.g., `"Fix header color on weekly template"`)
6. Click **Commit**, then **Push** (or use "Sync Changes")

### Recommended extensions

| Extension | What it's for |
|-----------|----------------|
| **Live Server** | Instantly preview an HTML file in your browser, updating as you type |
| **Prettier** | Auto-formats HTML/CSS so it stays clean and readable |
| **HTMLHint** (or similar) | Flags broken/invalid HTML before you push |

---

## 9. Rebuilding this system from scratch (step-by-step)

If you ever need to recreate this entire pipeline — for example, setting up a similar system for a different newsletter — here's the order to do it in.

### Step 1 — Create the GitHub repository

1. Create a new GitHub repo (e.g., `your-newsletter`)
2. Add two template files with placeholder text where dynamic content should go (e.g., `<!-- DLT_PLACEHOLDER -->`)
3. Add a `README.md` describing the project

### Step 2 — Get an OpenRouter account and API key

1. Sign up at OpenRouter
2. Generate an API key
3. Pick a model (we use `meta-llama/llama-3.3-70b-instruct` for cost and reliability — confirm current pricing/availability yourself, as model lineups change over time)

### Step 3 — Confirm your RSS sources work

Test every candidate feed manually with `curl` (see section 4) **before** building anything in Make. This single step prevents the majority of future headaches.

### Step 4 — Build the Make scenario

1. Create a new scenario in Make.com
2. Add a **Schedule trigger** module — set your desired time and days
3. Add an **HTTP > Get a file** module to fetch your template from GitHub's raw content URL (`raw.githubusercontent.com/...`)
4. Add one **HTTP** module per RSS feed, with "Parse response" enabled
5. Add an **OpenRouter (HTTP request)** module after each feed to generate a summary — pay close attention to field order (see section 10, rule 3)
6. Add a **SetVariable2** module that builds the final HTML by chaining `replace()` calls — one per placeholder
7. Add a **GitHub > Get a file** module to fetch the current file's SHA
8. Add a **GitHub > Update a file** module to push the finished HTML, referencing the SHA from the previous step

### Step 5 — Test thoroughly

1. Use **"Run once"** in Make to trigger a manual test
2. Check **Executions** in Make to confirm success
3. Check GitHub to confirm the file was updated correctly
4. Open the live URL to confirm it displays properly

### Step 6 — Set up the live website (Cloudflare Workers)

1. Create a Cloudflare account
2. Create a new Worker
3. Write a small script that fetches your HTML file from `raw.githubusercontent.com` and returns it to visitors
4. Deploy the Worker and note its public URL

### Step 7 — Turn on the schedule

Once you're confident everything works, activate the scenario so it runs automatically going forward.

---

## 10. Golden rules / best practices

These were all learned the hard way — following them will save you hours of debugging.

1. **Always verify an RSS feed manually before touching Make.** Use `curl` or your browser first. Sites that gate RSS behind a login can fail *silently* in Make if `stopOnHttpError` is `false`.

2. **Never let a Make "blueprint" exceed roughly 50–60,000 characters.** Going over this causes an Internal Server Error when updating the scenario. Large HTML files (40–50KB) must **never** be embedded as base64 text directly inside a Make module — instead, fetch them live from the raw GitHub URL at runtime.

3. **OpenRouter HTTP module field order matters.** The request must explicitly include the fields `shareCookies`, `parseResponse`, `allowRedirects`, `stopOnHttpError`, and `requestCompressedContent` — and `jsonStringBodyContent` must come **last**. Missing any of these, or putting them in the wrong order, produces a `BundleValidationError` and the whole run stops early.

4. **Inside SetVariable2, never nest `&` or `concat()` inside a `replace()` call.** It silently produces an `isinvalid: true` error. Instead, calculate any dynamic value (like a formatted date) in its own separate SetVariable2 module first, then reference it by module number later (e.g. `10.week_date`).

5. **GitHub's file fingerprint (SHA) for updates is always at `2.body.sha`** — never `2.sha` or `2.data.sha`. Getting this wrong causes a `422 Unprocessable Entity` error when pushing.

6. **OpenRouter's AI response array is 1-indexed, not 0-indexed.** The correct path is `module.data.choices[1].message.content`. If you omit the index, you get the whole array object instead of the text, which then base64-encodes incorrectly.

7. **Never trust the "success" message from the Run Once / scenarios_run button alone.** Always double-check the actual result in **Executions** — the button's own feedback can be misleading.

8. **Binary content from `raw.githubusercontent.com` needs `toString()` before you can run text operations on it.** Without it, `replace()` silently returns an empty string.

9. **Each newsletter section should pull from a different RSS source than its neighbors.** Otherwise the same article can appear in two different sections of the same newsletter.

10. **Always get approval before implementing a change — don't just go ahead.** Share candidate RSS sources, template changes, or scenario edits for sign-off first, especially for anything touching the live system.

---

## 11. Troubleshooting

### First step whenever something looks broken

1. Open Make → go to the scenario → click **History/Executions**
2. Find the most recent run and check its status
3. Click into it and open the **Logs** to see exactly where it failed

### Common errors and what they mean

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| HTTP 404 | The URL is wrong or the page moved | Re-check the RSS or GitHub URL |
| HTTP 403 | Missing or expired authentication | Check your GitHub token / OpenRouter API key |
| HTTP 422 (GitHub) | Wrong or missing SHA | Confirm you're reading `2.body.sha`, not `2.sha` |
| `"isinvalid": true` | A SetVariable2 expression has nested `&` or `concat()` | Simplify — calculate the value separately, reference it by module number |
| `BundleValidationError` | OpenRouter module is missing required fields or has them in the wrong order | Re-check field order (rule 3 above) |
| Scenario times out | Too many modules, or a slow external source | Reduce the number of modules per run, or check if a feed is responding slowly |
| Empty/blank sections in the final newsletter | `stopOnHttpError` is silently swallowing a broken feed | Manually test that specific RSS feed |
| Internal Server Error when saving the scenario | Blueprint size exceeded ~50-60KB | Move large HTML out of the scenario — fetch it at runtime instead |

---

## 12. Handover checklist

Use this when the project changes hands.

- [ ] New owner has access to Make.com (eu1.make.com) and understands the two active scenarios (6200212, 6220310) and the inactive one (6199804, must stay off)
- [ ] New owner has GitHub access to `MMaro06/montis-newsletter` (or the repo has been transferred/re-created under a new account)
- [ ] New owner has the OpenRouter API key (or has generated their own)
- [ ] New owner understands the Cloudflare Worker setup and has access to that account
- [ ] This guide and the `README.md` have both been reviewed
- [ ] Any pending template uploads (corrected HTML files) have been pushed to GitHub
- [ ] Unused/leftover files (e.g., debug scratch files) have been cleaned up, after confirming nothing else references them
- [ ] A test run of both scenarios has been done successfully with the new owner watching

---

**Good luck — and remember: verify first, approve second, implement third.** 🚀

*Maintained by Matteo Marochini — Montis Digital — Luxembourg*
