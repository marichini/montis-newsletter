# Montis Daily Digest — Handover Guide

> **Written by:** Matteo Marochini (m.marochini@montis.digital)
> **Last updated:** June 2026
> **Difficulty:** No coding required — everything runs through GUIs
> **Time to read:** ~15 minutes

---

## Table of Contents

1. [What this system does](#1-what-this-system-does)
2. [Architecture overview](#2-architecture-overview)
3. [Accounts & credentials you need access to](#3-accounts--credentials-you-need-access-to)
4. [GitHub — where the website lives](#4-github--where-the-website-lives)
5. [Cloudflare Workers — how the site is served](#5-cloudflare-workers--how-the-site-is-served)
6. [OpenRouter — the AI brain](#6-openrouter--the-ai-brain)
7. [Make — the automation engine](#7-make--the-automation-engine)
8. [The Make scenarios in detail](#8-the-make-scenarios-in-detail)
9. [The template files explained](#9-the-template-files-explained)
10. [How to verify the pipelines are working](#10-how-to-verify-the-pipelines-are-working)
11. [Common problems and how to fix them](#11-common-problems-and-how-to-fix-them)
12. [How to update the news sources](#12-how-to-update-the-news-sources)
13. [How to edit the newsletter design](#13-how-to-edit-the-newsletter-design)
14. [Cost & limits](#14-cost--limits)
15. [Emergency: how to manually trigger a newsletter](#15-emergency-how-to-manually-trigger-a-newsletter)
16. [Key technical gotchas (for developers)](#16-key-technical-gotchas-for-developers)
17. [Quick Reference Card](#17-quick-reference-card)

---

## 1. What this system does

Montis runs **two automated newsletters**, both powered by AI:

### Daily Digest
Every **weekday morning at 7:30 AM (Luxembourg time)**, the pipeline:
1. Fetches articles from institutional RSS feeds (DLT, regulation, post-trade, macro)
2. Sends those articles to an **AI model (Llama 3.3 70B via OpenRouter)** — six separate calls, one per newsletter section
3. The AI writes each section in HTML format, with real article links
4. Make **injects each AI-generated section** into the newsletter template
5. The final HTML is **committed to GitHub**
6. **Cloudflare Workers** automatically detects the new commit and serves the updated page

**Live URL:** `https://montis-newsletter.letzgomontis.workers.dev/daily-digest`

### Weekly Digest
Every **Monday morning at 7:30 AM**, a separate but structurally similar pipeline runs a longer, more curated newsletter with 7 sections covering the week's key themes.

**Live URL:** `https://montis-newsletter.letzgomontis.workers.dev/weekly-digest` *(check the repo for the exact filename)*

Both newsletters update automatically. The GitHub repo is the single source of truth — when Make commits an updated HTML file, the website updates within seconds.

**Running cost: ~$0.002/day** (less than €1/month) with the current AI model.

---

## 2. Architecture overview

Both pipelines follow the same pattern. Here is the Daily Digest flow:

```
7:30 AM (Mon–Fri)
        │
        ▼
  [Make Scheduler]  ←── Scenario ID: 6200212
        │
        ├──► RSS Feed: Ledger Insights        (DLT / institutional)
        └──► RSS Feed: PostTrade360           (post-trade / settlement)
                │
                ▼
     [6 × OpenRouter API calls]
     Model: Llama 3.3 70B, temperature 0.05
     One call per newsletter section:
       • TL;DR
       • Montis Radar
       • DLT & Tokenisation
       • Regulatory Pulse
       • Market Summary
       • Macro Outlook
                │
                ▼
     [GitHub GET] ← fetches daily-digest-template.html + SHA
                │
                ▼
     [SetVariable — 7-level nested replace() chain]
     Replaces all 7 placeholders with AI-generated HTML
                │
                ▼
     [GitHub PUT] ← commits daily-digest.html to the repo
                │
                ▼
     [Cloudflare Workers]
     Detects the new commit → serves the updated page live
```

The **Weekly Digest** (Scenario ID: 6220310) follows the same structure but uses different prompts, different placeholders, and commits `weekly-digest.html` instead.

---

## 3. Accounts & credentials you need access to

You need access to **4 services**. Credentials will be transferred directly — do not store passwords in plain text.

| Service        | URL                                   | What it's used for             | Login              |
|----------------|---------------------------------------|--------------------------------|--------------------|
| **GitHub**     | github.com/MMaro06/montis-newsletter  | Hosts all website files        | Personal account (MMaro06) |
| **Cloudflare** | dash.cloudflare.com                   | Serves the website publicly    | Montis email       |
| **Make**       | eu1.make.com                          | Runs both daily/weekly automations | Montis email   |
| **OpenRouter** | openrouter.ai                         | Pays for the AI API calls      | Personal account   |

> ⚠️ **Make free plan rule:** The free plan limits the number of **active scenarios** simultaneously. There are 3 scenarios in the account — keep only the two main ones active (IDs 6200212 and 6220310). The setup/debug scenario (ID 6199804) must stay **deactivated** at all times unless you specifically need it, and only while one of the main scenarios is temporarily paused.

---

## 4. GitHub — where the website lives

### Repository
**URL:** `https://github.com/MMaro06/montis-newsletter`

### Key files

| File                           | Role                                                            | Can you edit it?                              |
|--------------------------------|-----------------------------------------------------------------|-----------------------------------------------|
| `daily-digest-template.html`   | Daily newsletter skeleton — contains placeholders, never auto-modified | ✅ Only to change the design            |
| `daily-digest.html`            | The live daily page — overwritten every weekday morning by Make | ❌ Never edit manually                        |
| `weekly-digest-template.html`  | Weekly newsletter skeleton — same principle as daily template   | ✅ Only to change the design                  |
| `weekly-digest.html`           | The live weekly page — overwritten every Monday by Make         | ❌ Never edit manually                        |
| `index.html`                   | Homepage / landing page                                         | ✅ For design or content changes              |
| `worker.js`                    | Cloudflare Workers routing logic (maps URLs to files)           | ⚠️ Only if you change the URL structure       |

### How GitHub connects to Make

Make uses a **GitHub connection** named `"My GitHub connection (MMaro06)"` (connection ID: 8246786). This connection uses a Personal Access Token stored inside Make.

**If the connection breaks** (Make shows "GitHub 401 Unauthorized"):
1. Go to `github.com` → your profile → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
2. Click **"Generate new token (classic)"**
3. Give it a name (e.g. "Make newsletter"), select scope: **`repo`** (full repository access)
4. Copy the generated token immediately — it only shows once
5. In Make → left sidebar → **Connections** → find "My GitHub connection (MMaro06)" → click **Edit** → paste the new token → Save

### How to make design changes to the website

1. Edit `daily-digest-template.html` (or `weekly-digest-template.html`) locally or directly on GitHub
2. Commit the file to the repo (on GitHub: click the file → pencil icon → edit → "Commit changes")
3. The **next scheduled run** will use your updated template automatically
4. To preview immediately: trigger a manual run in Make (see Section 15)

> ⚠️ Never modify `daily-digest.html` or `weekly-digest.html` directly — Make overwrites them every run.

---

## 5. Cloudflare Workers — how the site is served

### Dashboard
`dash.cloudflare.com` → Workers & Pages → `montis-newsletter`

### How it works
- Cloudflare Workers is **connected to the GitHub repository** via a Pages deployment
- Every time Make commits a new HTML file to GitHub, Cloudflare detects the change and **automatically redeploys within ~30 seconds**
- No manual action is ever needed on Cloudflare during normal operation

### Live URLs
```
Daily Digest:  https://montis-newsletter.letzgomontis.workers.dev/daily-digest
Weekly Digest: https://montis-newsletter.letzgomontis.workers.dev/weekly-digest
```

### URL routing
The `worker.js` file in the repo handles routing. It maps each URL path (e.g. `/daily-digest`) to the corresponding HTML file. If you ever add a new page, you need to add a new route in `worker.js`.

### If the site goes down
1. Go to `dash.cloudflare.com` → Workers & Pages → `montis-newsletter` → **Deployments** tab
2. Look for a red/failed deployment — click it to see the error
3. Check the GitHub repo to confirm Make committed the file correctly (look at the last commit timestamp)
4. If a deployment is stuck, click any previous successful deployment → **"Rollback to this deployment"**

---

## 6. OpenRouter — the AI brain

### Dashboard
`https://openrouter.ai` → Log in → API Keys

### What it does
Make sends the raw RSS article text to OpenRouter, which forwards the request to **Llama 3.3 70B** — a powerful open-source AI model. The model reads the articles and writes each newsletter section as a clean HTML block.

### Where the API key is stored
The API key is stored inside Make as an HTTP Authorization header. You can find it in:

`Make → Scenario 6200212 → click any of the 6 "HTTP" (OpenRouter) modules → Headers tab → Authorization value`

The key starts with `sk-or-v1-...`

### Topping up credits
- Go to `openrouter.ai` → **Billing** → Add credits
- The pipeline costs ~$0.002/day, so **$5 lasts nearly a year**
- Recommended: enable **auto top-up** in billing settings (e.g. refill $5 when balance drops below $1) to avoid silent failures

### Swapping the AI model
The current model is `meta-llama/llama-3.3-70b-instruct`. If it ever becomes unavailable:
- Replacement option: `meta-llama/llama-3.1-70b-instruct` (same quality, same price range)
- To change: open any OpenRouter HTTP module in Make → find the `jsonStringBodyContent` field → change the `"model"` value

> ⚠️ **Google models (Gemini) are not available on this OpenRouter account.** Do not try to use them — they will fail silently.

---

## 7. Make — the automation engine

### Dashboard
`https://eu1.make.com` — always use the **eu1** subdomain (European data centre)

### The three scenarios

| Scenario ID | Name                                   | Status       | Role                                          |
|-------------|----------------------------------------|--------------|-----------------------------------------------|
| **6200212** | Montis Daily — AI Newsroom             | 🟢 Active    | Daily pipeline, runs Mon–Fri at 7:30          |
| **6220310** | Weekly Digest — Lundi 7h30             | 🟢 Active    | Weekly pipeline, runs Mondays at 7:30         |
| **6199804** | (Setup / Debug runner)                 | 🔴 Inactive  | One-time utility — keep deactivated           |

### Schedule configuration
Both active scenarios are configured for:
- **Time:** 07:30 AM
- **Timezone:** Europe/Luxembourg (CET in winter, CEST in summer — Make handles DST automatically)
- **Days:** Daily = Mon–Fri only / Weekly = Monday only

To view or edit the schedule: open the scenario → click the **clock icon** in the bottom-left of the canvas.

### Make operation quota (free plan)
The free plan includes **1,000 operations/month**. Each scenario run consumes approximately:
- Daily pipeline: ~15 operations/run × 22 working days = ~330 ops/month
- Weekly pipeline: ~13 operations/run × 4 Mondays = ~52 ops/month
- **Total: ~382 ops/month** — well within the 1,000 limit

If you ever need to upgrade (e.g. to add more scenarios), the Core plan at ~$9/month gives 10,000 operations.

---

## 8. The Make scenarios in detail

### 8a. Daily Digest — Scenario 6200212 (15 modules)

```
[Module 1]  Scheduler
            └─ Triggers at 7:30 Mon–Fri

[Module 2]  RSS Fetch — Ledger Insights
            └─ URL: https://ledgerinsights.com/feed/
            └─ Fetches the latest DLT/institutional articles

[Module 3]  RSS Fetch — PostTrade360
            └─ URL: https://www.posttrade360.com/feed/
            └─ Fetches post-trade and settlement articles

[Modules 4–9]  Six HTTP modules → OpenRouter API calls
            Each module:
            • Sends RSS content + a specific prompt to Llama 3.3 70B
            • Receives back one HTML section
            • Temperature: 0.05 (near-deterministic)
            One module per section:
              - TL;DR
              - Montis Radar
              - DLT & Tokenisation
              - Regulatory Pulse
              - Market Summary
              - Macro Outlook

[Module 10] GitHub GET
            └─ Fetches daily-digest-template.html
            └─ Returns: base64-encoded content + SHA

[Module 11] SetVariable (scope: roundtrip)
            └─ Decodes the template + runs 7 nested replace() calls
            └─ replace(replace(replace(replace(replace(replace(replace(
                 toString(toBinary(10.body.content; "base64")),
                 "DATE_PLACEHOLDER",   formatDate(now; "dddd, MMMM D, YYYY")),
                 "TLDR_PLACEHOLDER",   4.data.choices[1].message.content),
                 "MONTIS_PLACEHOLDER", 5.data.choices[1].message.content),
                 "DLT_PLACEHOLDER",    6.data.choices[1].message.content),
                 "REG_PLACEHOLDER",    7.data.choices[1].message.content),
                 "SUM_PLACEHOLDER",    8.data.choices[1].message.content),
                 "GEO_PLACEHOLDER",    9.data.choices[1].message.content)

[Modules 12–14]  (intermediate processing if applicable)

[Module 15] GitHub PUT
            └─ Commits daily-digest.html with the assembled HTML
            └─ Uses SHA from module 10 to avoid version conflicts
            └─ Commit message: "Daily digest update — YYYY-MM-DD"
```

### 8b. Weekly Digest — Scenario 6220310 (13 modules)

```
[Module 1]  Scheduler
            └─ Triggers at 7:30 on Mondays

[Module 2]  RSS Fetch — Ledger Insights
[Module 3]  RSS Fetch — PostTrade360

[Modules 4–9]  Six HTTP → OpenRouter calls
            One per section:
              - Week in Brief (BIG_PLACEHOLDER)
              - Compliance Corner (COMP_PLACEHOLDER)
              - Regulatory Watch (REG_PLACEHOLDER)
              - Tokenisation Spotlight (TOK_PLACEHOLDER)
              - Market Movers (MOV_PLACEHOLDER)
              - Must Read (READ_PLACEHOLDER)

[Module 10] SetVariable — week_date
            └─ Computes the week date string for display

[Module 11] SetVariable — final_html
            └─ 7-level replace() chain (same logic as daily)
            └─ References week_date as {{10.week_date}}
            └─ Uses WEEK_PLACEHOLDER (×3), BIG_PLACEHOLDER, COMP_PLACEHOLDER,
               REG_PLACEHOLDER, TOK_PLACEHOLDER, MOV_PLACEHOLDER, READ_PLACEHOLDER

[Module 12] GitHub GET
            └─ Fetches weekly-digest-template.html + SHA

[Module 13] GitHub PUT
            └─ Commits weekly-digest.html
```

### How OpenRouter modules are configured (important!)

Each OpenRouter HTTP module has a specific configuration that **must be preserved** if you ever recreate one:

| Field                    | Value                                          |
|--------------------------|------------------------------------------------|
| `shareCookies`           | `false`                                        |
| `allowRedirects`         | `true`                                         |
| `stopOnHttpError`        | `false`                                        |
| `requestCompressedContent` | `false`                                      |
| `jsonStringBodyContent`  | *(the full JSON prompt — must be placed LAST)* |
| `parseResponse`          | `true`                                         |

> ⚠️ The `jsonStringBodyContent` field **must always be last** in the mapper. Moving it breaks the module silently.

### How to read AI output from modules

The AI response is accessed in Make as:
```
{{MODULE_ID.data.choices[1].message.content}}
```
Use `[1]` (1-based indexing in Make). Using `choices[]` without an index returns an array object that encodes incorrectly.

---

## 9. The template files explained

### daily-digest-template.html

This file is the daily newsletter skeleton. It contains the full CSS, navbar, section structure, and **7 literal string placeholders** that Make replaces every morning:

| Placeholder          | Replaced with                                     |
|----------------------|---------------------------------------------------|
| `DATE_PLACEHOLDER`   | Today's date (e.g. "Monday, June 17, 2026")       |
| `TLDR_PLACEHOLDER`   | TL;DR section — AI-generated HTML                 |
| `MONTIS_PLACEHOLDER` | Montis Radar section — AI-generated HTML          |
| `DLT_PLACEHOLDER`    | DLT & Tokenisation section — AI-generated HTML    |
| `REG_PLACEHOLDER`    | Regulatory Pulse section — AI-generated HTML      |
| `SUM_PLACEHOLDER`    | Market Summary section — AI-generated HTML        |
| `GEO_PLACEHOLDER`    | Macro Outlook section — AI-generated HTML         |

### weekly-digest-template.html

Similar structure with **8 placeholders**:

| Placeholder          | Replaced with                                           |
|----------------------|---------------------------------------------------------|
| `WEEK_PLACEHOLDER`   | Week date string (appears 3× in the template)           |
| `BIG_PLACEHOLDER`    | Week in Brief section                                   |
| `COMP_PLACEHOLDER`   | Compliance Corner section                               |
| `REG_PLACEHOLDER`    | Regulatory Watch section                                |
| `TOK_PLACEHOLDER`    | Tokenisation Spotlight section                          |
| `MOV_PLACEHOLDER`    | Market Movers section                                   |
| `READ_PLACEHOLDER`   | Must Read section                                       |

### ⚠️ Critical rules about both templates

- **Never rename or remove a placeholder** — Make's `replace()` does exact string matching; one typo breaks the whole pipeline
- **Never modify `daily-digest.html` or `weekly-digest.html` directly** — they are overwritten every run
- **If you change the template design**, commit `daily-digest-template.html` (or weekly) to GitHub — the next scheduled run picks it up automatically
- **Large HTML changes** (structural redesigns) must be committed manually to GitHub via the web interface or git — they are too large to embed in Make blueprints (see Section 16)

---

## 10. How to verify the pipelines are working

### Daily check (30 seconds)

1. Visit `https://montis-newsletter.letzgomontis.workers.dev/daily-digest`
2. The **date in the header** should show today's date
3. Each article card should have a **real, clickable link** (not placeholder text)
4. No section should show the raw placeholder string (e.g. `TLDR_PLACEHOLDER`)

### Check Make execution history

1. Go to `eu1.make.com` → open Scenario 6200212 (or 6220310 for weekly)
2. Click **"History"** tab at the bottom
3. The most recent run should show a ✅ green status
4. If red, click on the failed run → identify which module failed → read the error message

### Check GitHub

1. Go to `github.com/MMaro06/montis-newsletter`
2. Click on `daily-digest.html` — the **"last committed"** timestamp should show this morning
3. If Make ran but GitHub wasn't updated, the GitHub PUT module failed (check Make history for the error)

---

## 11. Common problems and how to fix them

### ❌ The newsletter hasn't updated today

**Step 1 — Did Make run?**
- Open Make → Scenario 6200212 → History
- If no run appears: check the scenario is **Active** (the toggle in the scenario list should be blue/green). Check the schedule timezone.
- If it ran and failed: click the failed run → identify the red module → read the error.

**Step 2 — Check OpenRouter balance**
- Go to `openrouter.ai` → Billing
- If balance is $0, the AI calls fail with a 402 error. Top up.

**Step 3 — Check GitHub**
- Was the file committed? Check `daily-digest.html` last commit timestamp.
- If not: the GitHub PUT module failed. Most likely cause: the SHA in the PUT doesn't match the current file's SHA (conflict). Fix: run the scenario again manually.

---

### ❌ The page shows garbled content or raw HTML tags

Usually means the AI returned something unexpected — markdown fences (` ```html `), extra commentary, or incomplete HTML.

**Quick fix:**
1. Go to `github.com/MMaro06/montis-newsletter`
2. Click `daily-digest.html` → click **"History"** (top right)
3. Find the last good version → click its commit hash → click **"Revert"**
4. The next morning's run will overwrite it with fresh content

**Root cause investigation:**
- Open Make → failed run history → check the OpenRouter module output
- If the AI added markdown fences: the prompt needs reinforcing with "output ONLY the HTML, no backticks, no preamble, no explanation"
- Temperature is already 0.05 — do not lower it further

---

### ❌ Make shows "GitHub 401 Unauthorized"

The GitHub Personal Access Token has expired or been revoked.

**Fix:**
1. Go to `github.com` → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
2. Generate a new token → scope: `repo` → copy it immediately
3. In Make → left sidebar → **Connections** → "My GitHub connection (MMaro06)" → **Edit** → paste the new token

---

### ❌ Make shows "402 Payment Required" on an OpenRouter module

**Fix:** Top up OpenRouter credits at `openrouter.ai` → Billing. Then re-run the scenario manually.

---

### ❌ Make shows "429 Too Many Requests" on an OpenRouter module

Rate limit hit — rare with the current setup but can happen if modules run too close together.

**Fix:** Add a **"Sleep" module** between two OpenRouter HTTP modules:
1. Open the scenario → click the `+` between two HTTP modules
2. Search for "Sleep" → add it → set duration to **2 seconds**
3. Save and retest

---

### ❌ Placeholder text appears literally on the page (e.g. "DATE_PLACEHOLDER")

The `replace()` chain in SetVariable failed. Causes:
- The GitHub GET module didn't return the template correctly
- The placeholder was accidentally renamed in the template file

**Debug steps:**
1. Open Make → run history → click the failed run
2. Click the GitHub GET module (module 10 for daily) → inspect the output
3. `body.content` should be a long base64 string — if it's empty or an error, the GET failed
4. Check the template file exists on GitHub and hasn't been accidentally deleted

---

### ❌ Make "Internal Server Error" when saving a scenario

This is a known Make API limitation — the blueprint JSON cannot exceed ~35–40 KB.

**This happens when:** you try to embed large content (base64-encoded HTML, very long prompts) directly in the blueprint.

**Fix:** Never embed large strings in the blueprint directly. Instead:
- Store templates on GitHub and fetch them at runtime via an `http:MakeRequest` module
- Always edit scenarios through the Make **visual web editor**, not via the API

---

### ❌ The weekly digest didn't run on Monday

1. Check Scenario 6220310 is **Active** in Make
2. Verify the schedule: open the scenario → clock icon → confirm it's set to Monday, 07:30, Europe/Luxembourg
3. Check Make operation quota: `eu1.make.com` → left sidebar → **Usage** — if you're at 1,000 ops/month, runs are blocked until the month resets

---

## 12. How to update the news sources

### To change an RSS feed

1. Go to `eu1.make.com` → Scenario 6200212 (or 6220310) → **Edit**
2. Click the RSS module you want to change
3. Update the **URL** field to the new feed
4. Update the **AI prompt** in the paired OpenRouter HTTP module to mention the new source name
5. Click **Save** → test with **"Run once"**

### Current RSS feeds in use

| Pipeline | Section           | Feed URL                                      |
|----------|-------------------|-----------------------------------------------|
| Both     | DLT / General     | `https://ledgerinsights.com/feed/`            |
| Both     | Post-trade        | `https://www.posttrade360.com/feed/`          |

> To see all active feed URLs: open the scenario → click each RSS module → check the URL field.

### Recommended alternative institutional feeds

| Source         | Feed URL                                      | Best for                        |
|----------------|-----------------------------------------------|---------------------------------|
| ESMA           | `https://www.esma.europa.eu/rss.xml`          | EU regulation, CSDR, MiCA       |
| ECB            | `https://www.ecb.europa.eu/rss/press.html`    | Monetary policy, wholesale CBDC |
| The Banker     | `https://www.thebanker.com/rss`               | Banking, regulation             |
| Risk.net       | `https://www.risk.net/rss`                    | Risk, derivatives               |
| BIS            | `https://www.bis.org/rss/index.htm`           | Central banking, settlement      |
| FT Markets     | `https://www.ft.com/markets?format=rss`       | Macro (check FT subscription)   |

---

## 13. How to edit the newsletter design

The newsletter design lives entirely in the template HTML files. You don't need to touch Make at all for visual changes.

### Minor edits (text, colours, labels)
1. Go to `github.com/MMaro06/montis-newsletter`
2. Click `daily-digest-template.html` → click the **pencil icon** (Edit)
3. Make your changes
4. Click **"Commit changes"** with a short message (e.g. "Update header colour")
5. The next morning's Make run will use the new template

### Major edits (section structure, new CSS)
1. Download the file locally, edit it with a code editor (VS Code, etc.)
2. Test it by opening the HTML file in your browser
3. Upload it back to GitHub: click "Add file" → "Upload files" → drag and drop → Commit

### Adding a new section to the newsletter
This requires changes in both the template and Make:
1. Add a new placeholder (e.g. `NEW_PLACEHOLDER`) in the template HTML where you want the section
2. In Make → Scenario 6200212 → add a new OpenRouter HTTP module with a prompt for that section
3. In the SetVariable module, add one more `replace()` level wrapping the existing chain:
   ```
   replace(...existing chain..., "NEW_PLACEHOLDER", {{NEW_MODULE.data.choices[1].message.content}})
   ```
4. Save and test with "Run once"

> ⚠️ Keep the number of nested `replace()` calls in mind — Make handles up to ~10 levels reliably.

---

## 14. Cost & limits

### Monthly cost breakdown

| Service                              | Cost                          |
|--------------------------------------|-------------------------------|
| OpenRouter — Llama 3.3 70B           | ~$0.002/day → **~€0.70/month** |
| Make (free plan)                     | **$0** — 1,000 ops/month       |
| GitHub (public repo, free plan)      | **$0**                        |
| Cloudflare Workers (free tier)       | **$0** — up to 100k req/day   |
| **Total**                            | **< €1/month**                |

### Make operation usage

| Pipeline        | Ops/run | Runs/month | Ops/month |
|-----------------|---------|------------|-----------|
| Daily Digest    | ~15     | ~22        | ~330      |
| Weekly Digest   | ~13     | ~4         | ~52       |
| **Total**       |         |            | **~382**  |

Free plan limit: 1,000 ops/month → **618 ops of headroom** remaining.

### When to upgrade Make
Upgrade to the Core plan (~$9/month, 10,000 ops) only if you:
- Add significantly more modules per scenario
- Add additional scenarios
- Approach the 1,000 ops/month limit

---

## 15. Emergency: how to manually trigger a newsletter

If you need to publish outside the normal 7:30 schedule (after a failure, to test a change, etc.):

1. Go to `eu1.make.com` → open the relevant scenario (6200212 for daily, 6220310 for weekly)
2. Click **"Run once"** — the play button in the bottom-left corner of the canvas
3. Wait **60–90 seconds** for all modules to complete
4. Check the live URL — it should reflect the new content

> ⚠️ Each manual run counts toward your monthly Make operation quota (~15 ops for daily, ~13 for weekly).

To **test without publishing** (e.g. to debug a module), you can right-click individual modules in the scenario canvas and click "Run this module only" — but be aware that modules requiring output from earlier modules will fail without their input.

---

## 16. Key technical gotchas (for developers)

This section is for anyone who needs to rebuild or significantly modify the pipeline from scratch.

### `toString()` is required for GitHub content
Raw content fetched from `raw.githubusercontent.com` is returned as binary. Before running `replace()` on it, always wrap it in `toString()`:
```
toString(toBinary(10.body.content; "base64"))
```
Skipping `toString()` causes `replace()` to return an empty string silently.

### GitHub API response envelope
All GitHub API responses via `github:makeRestApiCall` are wrapped in a `body` envelope. The SHA for PUT operations is at:
```
{{GITHUB_GET_MODULE.body.sha}}
```
Not `2.sha` or `2.data.sha` — it's always `body.sha`.

### OpenRouter response path (1-based indexing)
With `parseResponse: true`, the AI model's response is at:
```
{{MODULE.data.choices[1].message.content}}
```
Make uses **1-based array indexing**. Using `choices[]` without an index returns an array object that base64-encodes incorrectly when passed downstream.

### Blueprint size limit
Make's `scenarios_update` API fails with an Internal Server Error when the blueprint JSON exceeds ~35–40 KB. This means you **cannot embed large base64 strings** (like a full HTML template) directly in the blueprint. The working solution: fetch the template from raw GitHub at runtime via `http:MakeRequest`.

### OpenRouter module field ordering
The `jsonStringBodyContent` field in OpenRouter HTTP modules **must be placed last** in the mapper. The order must be:
1. `shareCookies`
2. `allowRedirects`
3. `stopOnHttpError`
4. `requestCompressedContent`
5. `jsonStringBodyContent` ← always last

### Weekly digest week_date reference
In the weekly scenario, the computed week date (module 10) is referenced in the SetVariable chain as `{{10.week_date}}`, not `{{10.value}}`. This is because it's set via a named SetVariable module, not a generic variable.

---

## 17. Quick Reference Card

```
═══════════════════════════════════════════════════════════════
  MONTIS NEWSLETTER — QUICK REFERENCE
═══════════════════════════════════════════════════════════════

LIVE PAGES
  Daily:    https://montis-newsletter.letzgomontis.workers.dev/daily-digest
  Weekly:   https://montis-newsletter.letzgomontis.workers.dev/weekly-digest

KEY PLATFORMS
  GitHub:      https://github.com/MMaro06/montis-newsletter
  Make:        https://eu1.make.com  (use eu1 subdomain!)
  Cloudflare:  https://dash.cloudflare.com → Workers & Pages → montis-newsletter
  OpenRouter:  https://openrouter.ai → Billing (top up if balance < $1)

MAKE SCENARIOS
  6200212 → Daily Digest     → Mon–Fri, 07:30 → 🟢 Keep ACTIVE
  6220310 → Weekly Digest    → Mondays, 07:30 → 🟢 Keep ACTIVE
  6199804 → Setup/Debug      → on demand      → 🔴 Keep INACTIVE

TEMPLATE FILES (edit these for design changes)
  daily-digest-template.html    ← 7 placeholders
  weekly-digest-template.html   ← 8 placeholders

LIVE FILES (never edit manually — overwritten by Make)
  daily-digest.html
  weekly-digest.html

DAILY PLACEHOLDERS
  DATE_PLACEHOLDER, TLDR_PLACEHOLDER, MONTIS_PLACEHOLDER,
  DLT_PLACEHOLDER, REG_PLACEHOLDER, SUM_PLACEHOLDER, GEO_PLACEHOLDER

WEEKLY PLACEHOLDERS
  WEEK_PLACEHOLDER (×3), BIG_PLACEHOLDER, COMP_PLACEHOLDER,
  REG_PLACEHOLDER, TOK_PLACEHOLDER, MOV_PLACEHOLDER, READ_PLACEHOLDER

AI MODEL
  meta-llama/llama-3.3-70b-instruct via OpenRouter
  Temperature: 0.05 | Cost: ~$0.002/day

RSS SOURCES (currently active)
  Ledger Insights:  https://ledgerinsights.com/feed/
  PostTrade360:     https://www.posttrade360.com/feed/

GITHUB CONNECTION IN MAKE
  Name: "My GitHub connection (MMaro06)"
  Connection ID: 8246786
  If broken: regenerate GitHub PAT with "repo" scope → reconnect in Make

═══════════════════════════════════════════════════════════════
```

---

*Written during the transition period in June 2026. The Make execution history and GitHub commit log are your two best debugging tools — they record exactly what happened at every step of every run. When in doubt, check those first.*
