# TeamBrief
TeamBrief turns messy standup updates into structured Done / Next / Blockers, highlights recurring/aging/high-impact blockers, and exports a clean team standup to Slack/Notion Markdown.

# TeamBrief

TeamBrief turns messy standup updates into clean **Done / Next / Blockers**, highlights **recurring + aging + high-impact (blast radius)** blockers with clear categories, and exports a polished team standup to **Slack/Notion Markdown**.

Built as a 5-day hackathon MVP (Next.js + Supabase + Prisma + OpenAI).

---

## Why this exists

Daily standups are often slow and low-signal because updates are inconsistent:
- “worked on backend”
- “still on it”
- “blocked” (blocked on what?)

TeamBrief standardizes updates automatically and makes blockers visible early — especially the ones that keep coming back and affect multiple people.

---

## What it does (MVP)

### 1) One message → standup format
Paste one update like:

> "Merged PR #42 fixing login validation. Today I’ll add unit tests. Blocked: payment sandbox credentials expired."

TeamBrief extracts:
- **Done:** Merged PR #42 fixing login validation
- **Next:** Add unit tests
- **Blockers:** Payment sandbox credentials expired

### 2) Quality improvement (built-in)
If an item is vague or overloaded, TeamBrief suggests rewrites (you can accept or edit).

### 3) Today Standup view
Aggregates all members into a clean daily standup with strict display caps per person:
- Done ≤ 3
- Next ≤ 3
- Blockers ≤ 2  
(If someone submits multiple times, we store all updates but display the most recent items first.)

### 4) Blocker Board (the differentiator)
Clusters similar blockers and shows:
- **Recurring** blockers (clustered)
- **Aging** (how long it’s been open)
- **Blast radius** (how many people are affected)
- **Category**: Access / Decision / Dependency / Technical / External / Capacity
- **Status**: Open / Resolved + “Resurfaced” indicator when a resolved blocker returns

### 5) Export (copy/paste)
Exports the **project-level Today Standup** to:
- **Slack format** (mrkdwn-friendly)
- **Notion format** (standard Markdown)

---

## Demo flow (what we show judges)
1. Create/select a project and enter your name (no login)
2. Paste messy updates (simulate 4–6 team members)
3. Confirm extracted Done/Next/Blockers
4. Show Today Standup (clean output)
5. Show Blocker Board (recurring + aging + blast radius)
6. Export to Slack/Notion Markdown (copy)

---

## Tech stack
- **Next.js** (fullstack)
- **Supabase Postgres**
- **Prisma**
- **Vercel** deployment
- **OpenAI API**
  - `gpt-4o-mini` (extraction, categorization, rewrite suggestions)
  - `text-embedding-3-small` (embeddings similarity for clustering fallback)

---

## Project structure
```txt
src/
  app/                 # Next.js routes + UI
  modules/
    projects/          # create/select projects
    updates/           # extraction + quality + submit
    standups/          # daily aggregation + export
    blockers/          # categorize + normalize + cluster + status
  shared/
    ai/                # AI gateway + prompts + schemas
    utils/             # normalization + similarity helpers
prisma/
