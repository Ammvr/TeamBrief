# TeamBrief — Product Requirements Document

**Version 2.0 — Revised | March 2026 | Hackathon MVP**

**Status:** Ready for Development
**Stack:** Next.js, Supabase, Prisma, Vercel, OpenAI

---

## 1. Product Summary

TeamBrief converts a single messy daily update (free-form text) into a structured standup organized as Done / Next / Blockers. It generates a project-level daily view, provides an intelligent Blocker Board that highlights Recurring, Aging, and Blast Radius blockers with category tags, and exports the standup as Markdown for Slack or Notion.

**Authentication:** None required for MVP. Users enter a Name and select or create a Project. A locally-generated `deviceId` (UUID) is stored in localStorage and sent with every submission to enable conflict detection and future auth migration.

---

## 2. Goals

- Produce a clean, structured standup from messy free-form updates in seconds.
- Make blockers actionable by surfacing: Recurring clusters, Aging duration, Blast Radius (affected members), and Category tags.
- Be reliable and fast on Vercel (low failure rate, snappy responses with well-defined latency targets).
- Enforce update quality through a two-tier validation system (hard fail / soft fail) without slowing down the standup flow.

---

## 3. Non-Goals

- Voice note upload / transcription.
- Slack bot posting (OAuth), Jira / Asana integrations.
- Multi-workspace org admin / RBAC.
- Batch-processing many updates simultaneously in a single user action.
- Per-person or Blocker Board-only export modes (MVP exports project-level standup only).

---

## 4. Target Users

Mixed project teams (Engineering / Product / Design) doing daily standups. Project managers and hackathon teams who need fast, structured status communication.

---

## 5. Core User Flows

### Flow A — Submit Update

1. User selects or creates a Project.
2. User enters Name (stored locally alongside `deviceId`).
3. User writes one free-form message update.
4. Client-side input gate validates minimum quality (see Section 12).
5. System runs AI-1 extraction → server runs quality checks → AI-3 fires if flagged → shows Done / Next / Blockers in a confirmation screen.
6. Flagged items display inline with `reviewReason` and rewrite suggestions (hard fails block submit; soft fails are dismissible).
7. User edits / accepts → Submit.
8. App updates: Today Standup view is immediate; Blocker Board updates asynchronously.

### Flow B — View Standup

User opens the Today view and sees a team-level Done / Next / Blockers summary aggregated from all submitted updates for the selected project and date. A "View Blockers" link navigates to the Blocker Board.

### Flow C — Export

User clicks Export Markdown, selects format (Slack or Notion) and view (Team Summary or By Person), then copies via a one-click copy button.

---

## 6. Core Features (MVP)

### 6.1 Projects

- Create and select multiple projects.

### 6.2 One-Message Update Input

- Single free-form text input per submission.
- Multiple submissions per day are allowed; entries are appended (preserves full history).
- Display caps (Done ≤ 3, Next ≤ 3, Blockers ≤ 2) apply to the Today Standup view per person, not to storage.

### 6.3 AI Extraction + Confirmation

- Extract raw text into Done / Next / Blockers with bounded counts.
- Confirmation screen for quick edits before final submission.
- If `next[]` or `blockers[]` are empty, show non-blocking nudges with one-tap actions: **Add Next**, **Add Blocker**, **Confirm None**.
- Items with `reviewReason` are highlighted for user attention.

### 6.4 Today Standup View

- Aggregated Done / Next / Blockers for the selected project and date.
- Per-person display caps applied (latest N items shown per category).
- Lightweight "View Blockers" link to the Blocker Board.

### 6.5 Blocker Board

- **Recurring:** cluster similar blockers across days using semantic embeddings.
- **Categories:** Access / Decision / Dependency / Technical / External / Capacity.
- **Aging:** open duration per cluster (days since `firstSeenAt`).
- **Blast Radius:** count of unique members affected per cluster.
- **Status:** Open / Resolved with manual toggle.
- **Resurfaced badge:** shown on cluster cards when `resurfacedCount > 0` and `lastResurfacedAt` is within the board range (e.g., last 7 days). Tooltip shows resolution and resurfacing history.
- May show a brief "Updating clusters…" spinner while async processing completes.

### 6.6 Markdown Export

- Standard Markdown core with two lightweight formatting wrappers:
  - **Notion:** real Markdown headings and bullets (`## Done`, `- item`).
  - **Slack:** bold labels and bullets (`*Done*` then `• item`).
- View toggle inside export modal: **Team Summary** (default) or **By Person** (same data, grouped).
- Compact Blocker Highlights footer: top 3 recurring/aging clusters with category, age, and blast radius.
- One-click copy-to-clipboard button.

---

## 7. AI Usage

All AI tasks use **gpt-4o-mini** for speed and cost efficiency. Structured Outputs / strict schema mode is used for AI-1 and AI-3 to ensure machine-parseable JSON responses.

### 7.1 AI-1: Update Extraction (Required)

- **Trigger:** After the user submits the one-message update.
- **Output:** Strict JSON `{ done[], next[], blockers[] }` with max sizes (3/3/2).
- **Policy:** In-progress statements (e.g., "working on X") are classified as **Next**, not Done. Done is reserved for completed outcomes.
- **Constraint:** Must not invent details; only rewrite/split what the user provided. Include instruction "do not add new facts" in the system prompt.
- **Review flag:** If classification confidence is low, add `reviewReason: string` on the item (e.g., "Marked as Next — sounds in-progress, not completed").

### 7.2 AI-2: Blocker Categorization (Required)

- **Trigger:** After extraction, for each blocker (batched per update). Runs asynchronously post-submit.
- **Output:** One category from the fixed set: Access / Decision / Dependency / Technical / External / Capacity.

### 7.3 AI-3: Quality Rewrite Suggestions (Conditional)

- **Trigger:** Only when server-side quality checks flag an item as a hard fail (see Section 9).
- **Output:** 2–3 rewrite suggestions per flagged item; user must choose or edit to clear the flag.
- **Soft fails:** Suggestions are shown but submission is not blocked.

### 7.4 Embeddings: Blocker Clustering

- **Model:** text-embedding-3-small with reduced dimensions (256) for cost and storage efficiency.
- **Trigger:** For each new blocker, after AI-2 categorization. Runs asynchronously.
- **Method:** Generate embedding, compare via cosine similarity against existing cluster centroids within the same project and category. Cluster if above threshold.

---

## 8. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | User can create/select a project. |
| FR-2 | User can submit a one-message update (free-form text). |
| FR-3 | System extracts Done/Next/Blockers via AI-1 and displays a confirmation UI with quality flags and `reviewReason` annotations. |
| FR-4 | Multiple submissions per day allowed; entries are appended (preserves full history). Display caps (FR-5) apply at the view layer only. |
| FR-5 | Display limits: Done ≤ 3, Next ≤ 3, Blockers ≤ 2 per person in the Today Standup view. Server stores all items without truncation. |
| FR-6 | Store: `rawText`, extracted arrays, `timestamp`, `memberName`, `date`, `deviceId`, `extractionStatus`, `clusteringStatus`, `qualityScore`. |
| FR-7 | Categorize each blocker into one of 6 categories via AI-2 (async post-submit). |
| FR-8 | Blocker clustering via text-embedding-3-small cosine similarity. Each cluster tracks `firstSeenAt`, `lastSeenAt`, `count`, `uniqueMemberCount`. |
| FR-9 | Aging computed as `today − firstSeenAt` in whole days. |
| FR-10 | Blast radius computed as number of distinct `memberName`s in the cluster. |
| FR-11 | Cluster status: default Open; user can mark Resolved (stores `resolvedAt`). If a resolved cluster gets a new matching blocker: set `status=Open`, increment `resurfacedCount`, set `lastResurfacedAt=now`, keep `resolvedAt` for history. |
| FR-12 | Export generates Markdown (Slack/Notion wrappers) with team summary/by-person toggle and Blocker Highlights footer. Supports copy-to-clipboard. |
| FR-13 | Generate `deviceId` (UUID) on first visit, persist in localStorage, send with every submission. Show non-blocking warning if same `memberName` appears from different `deviceId`s within a project. |
| FR-14 | Input gate rejects submissions with fewer than 8 characters, fewer than 2 words, no letters, or only stopwords. Friendly error message displayed. |

---

## 9. Quality / Normalization

Quality checks run **server-side on extracted items** (after AI-1), not on raw text. The pipeline is:

1. AI-1 extracts `{ done[], next[], blockers[] }`
2. Server runs `qualityCheck()` on each item
3. If hard-fail flags exist → call AI-3 for suggestions
4. Return to UI: extracted items + flags + suggestions
5. User edits/accepts → submit → final server validation (same checks)

### 9.1 Two-Tier Severity System

#### Hard Fail (Blocking)

Submission is blocked until the user picks an AI-3 suggestion or edits the item to clear the flag.

- **`vague_severe`:** "worked on backend", "fixed bugs", "did stuff", "progress"
- **`multi_task_severe`:** 3+ actions in one bullet (detected via 2+ conjunctions: and, then, also, plus, &)
- **`no_action`:** No verb present ("login page", "backend", "meeting")
- **`too_long`:** Paragraph-length item that cannot be summarized cleanly

#### Soft Fail (Non-Blocking)

Warnings and suggestions shown, but user can dismiss and submit.

- Slightly vague but usable ("fixed login issue").
- Two actions in one bullet but understandable ("refactored auth and added tests").
- Missing next/blockers (nudges as described in Section 6.3).

### 9.2 Multi-Task Detection Heuristic

- Contains 2+ conjunctions: `and`, `then`, `also`, `plus`, `&`.
- Contains 2+ separators: semicolons or commas with multiple verbs.
- **Severity:** 3+ actions = hard fail; 2 actions = soft fail.

### 9.3 Quality Score

A per-member `qualityScore` (0–100) is computed as a rolling average over the last N submissions, based on soft/hard fail rates. Stored on each Update record. Not exposed to users in MVP; available for future use (e.g., nudges, selective AI-3 skipping for high-quality contributors).

---

## 10. Data Model

### Project

| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | String | Project name |
| `createdAt` | DateTime | Auto-generated |

### Update

| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID | Primary key |
| `projectId` | UUID (FK) | References Project.id |
| `memberName` | String | User-entered name |
| `deviceId` | UUID | Generated on first visit, stored in localStorage |
| `createdAt` | DateTime | Submission timestamp |
| `date` | Date | Standup date (YYYY-MM-DD) |
| `rawText` | Text | Original user input |
| `doneJson` | JSON | Array of done items with optional `reviewReason` |
| `nextJson` | JSON | Array of next items with optional `reviewReason` |
| `blockersJson` | JSON | Array of blocker items with optional `reviewReason` |
| `extractionStatus` | Enum | `SUCCESS` \| `FAILED` \| `MANUAL` |
| `clusteringStatus` | Enum | `PENDING` \| `COMPLETED` \| `FAILED` |
| `qualityScore` | Integer | 0–100, rolling average of quality checks |

### BlockerItem

| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID | Primary key |
| `updateId` | UUID (FK) | References Update.id |
| `projectId` | UUID (FK) | References Project.id |
| `memberName` | String | Denormalized for query performance |
| `date` | Date | Standup date |
| `text` | String | Original blocker text |
| `category` | Enum | Access \| Decision \| Dependency \| Technical \| External \| Capacity |
| `embedding` | Vector(256) | text-embedding-3-small, reduced dimensions |
| `clusterId` | UUID (FK) | References BlockerCluster.id |

### BlockerCluster

| Field | Type | Notes |
|-------|------|-------|
| `id` | UUID | Primary key |
| `projectId` | UUID (FK) | References Project.id |
| `category` | Enum | Same enum as BlockerItem |
| `representativeText` | String | Human-readable cluster label |
| `centroidEmbedding` | Vector(256) | Average embedding of cluster members |
| `count` | Integer | Number of blocker items in cluster |
| `firstSeenAt` | Date | Earliest blocker date in cluster |
| `lastSeenAt` | Date | Most recent blocker date |
| `uniqueMemberCount` | Integer | Distinct memberNames (blast radius) |
| `status` | Enum | `OPEN` \| `RESOLVED` |
| `resolvedAt` | DateTime? | Nullable; kept on resurface for history |
| `resurfacedCount` | Integer | Default 0; incremented on each resurface |
| `lastResurfacedAt` | DateTime? | Nullable; set when cluster resurfaces |

---

## 11. Clustering Approach

Semantic embedding-based clustering replaces the original Jaccard token-overlap method for significantly improved accuracy on short, varied blocker descriptions.

### 11.1 Pipeline

1. Generate a 256-dimension embedding for the new blocker text using text-embedding-3-small.
2. Query existing clusters within the same project and same category.
3. Compute cosine similarity between the new embedding and each cluster centroid.
4. If similarity exceeds the threshold: assign to the best-matching cluster, update the centroid (running average), and update cluster metadata.
5. If no match: create a new cluster with this blocker as the seed.
6. If the matched cluster has `status=RESOLVED`: set `status=OPEN`, increment `resurfacedCount`, set `lastResurfacedAt=now`.

### 11.2 Design Decisions

- Clustering is scoped to **same project + same category** to reduce false positives.
- Cosine similarity threshold should be tuned during development (recommended starting point: **0.82**).
- Centroid is a running average of all member embeddings in the cluster.
- This approach runs **asynchronously post-submit** and does not block the user.

---

## 12. Error Handling

### 12.1 AI-1 Extraction Failure

1. Attempt AI-1 call.
2. On failure: retry once with 800–1200ms jitter backoff.
3. If retry fails: show manual fallback screen (empty Done / Next / Blockers fields + raw text visible). User submits manually. Store with `extractionStatus = FAILED`.

### 12.2 Malformed AI Output

- Strict server-side schema validation using **Zod**: required arrays, cap enforcement (3/3/2), string length limits (160 chars), strip empty/duplicate items.
- If invalid: one repair attempt (re-prompt AI with "return valid JSON only").
- If still invalid: manual fallback UI.

### 12.3 Input Gate (Pre-AI Validation)

Reject and show a friendly error if the input has fewer than 8 characters, fewer than 2 words, no letters, or only stopwords.

**Message:** "Your update is too short to extract. Add 1–2 sentences about what you did, what's next, and any blockers."

### 12.4 Async Pipeline Failure (AI-2 / Clustering)

- Store `clusteringStatus = FAILED` on the Update record.
- Blocker Board shows the item as uncategorized/unclustered.
- Retry mechanism can be triggered manually or via a background job.

---

## 13. Architecture

### 13.1 Pipeline Split

The submission pipeline is split into a **critical path** (user-blocking) and a **background path** (async) to optimize perceived performance.

> **Critical Path (blocks user) — target ≤ 3s p95**
> AI-1 extraction → server-side quality checks → AI-3 rewrite suggestions (if flagged) → confirmation UI

> **Background Path (async, post-submit) — target ≤ 5s best-effort**
> AI-2 blocker categorization → embedding generation → clustering → Blocker Board update

After submit, the user is redirected to the Today Standup view immediately. The Blocker Board may show a brief "Updating clusters…" spinner while background processing completes.

### 13.2 Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | Next.js (App Router, fullstack) |
| Database | Supabase Postgres |
| ORM | Prisma |
| Hosting | Vercel |
| AI (extraction, categorization, suggestions) | OpenAI gpt-4o-mini with Structured Outputs |
| Embeddings | OpenAI text-embedding-3-small (256 dimensions) |
| Schema validation | Zod |

---

## 14. Performance Targets

Updates are expected to be submitted throughout the day by different members. The system is not optimized for batch-processing many updates simultaneously in a single user action for MVP.

| Metric | Description | Target |
|--------|-------------|--------|
| Metric A | Per submission: from paste to extracted fields + suggestions shown | ≤ 3s p95 |
| Metric B | Post-submit: blocker categorization + clustering visible on Blocker Board | ≤ 5s best-effort |
| Metric C | Today Standup view render (DB query + formatting) | ≤ 500ms server |

### 14.1 Hackathon Demo Validation

- 4–6 messy updates submitted sequentially → extracted, confirmed, and aggregated into Today Standup within performance targets.
- Blocker Board correctly shows at least 1 recurring cluster with accurate aging (days) and blast radius (unique members).
- Self-reported prep time reduction ≥ 50% in quick test.

> **Demo Tip: Seed Demo Updates**
> Optional "Seed demo updates" button that inserts pre-extracted updates directly (no AI calls) and pre-creates blocker clusters. Gives instant board population for judge reliability.

---

## 15. Identity & Data Retention

### 15.1 Device Identity

- A `deviceId` (UUID) is generated on first visit and persisted in localStorage.
- Sent with every submission and stored on Update records.
- If the same `memberName` appears from different `deviceId`s within a project, a non-blocking warning is shown: "This name is used on another device. Consider adding an initial (e.g., Ammar S)."
- Enables future migration: `deviceId` → user account mapping.

### 15.2 Data Retention

Data is retained indefinitely for the duration of the hackathon demo. A manual "Delete Project" action may be added post-hackathon. Optional automated cleanup (e.g., delete projects inactive for 30 days) is out of scope for MVP.

---

## 16. Revision Log (v1.0 → v2.0)

| Area | Change |
|------|--------|
| Clustering (S11) | Replaced Jaccard token-overlap with semantic embeddings (text-embedding-3-small, cosine similarity) |
| FR-4 / FR-5 | Clarified: append freely, display caps apply at view layer only |
| FR-11 / Blocker Board | Defined resurfaced behavior: Open + badge (`resurfacedCount`, `lastResurfacedAt`), no distinct state |
| AI Usage (S7) | Added `reviewReason` flag, in-progress → Next policy, specified gpt-4o-mini, added embedding model |
| Quality (S9) | Two-tier severity system (hard fail blocking / soft fail non-blocking), server-side checks on extracted items |
| Data Model (S10) | Added: `deviceId`, `extractionStatus`, `clusteringStatus`, `qualityScore`, `resurfacedCount`, `lastResurfacedAt`, `embedding`, `centroidEmbedding` |
| Error Handling (S12) | New section: retry policy, schema validation (Zod), input gate, manual fallback, async failure handling |
| Architecture (S13) | New section: critical path vs background pipeline split, async AI-2/clustering |
| Performance (S14) | Replaced 10s bulk metric with three-tier targets (3s/5s/500ms), clarified staggered submission model |
| Export (S6.6) | Defined: standard Markdown + Slack/Notion wrappers, team summary/by-person toggle, Blocker Highlights footer |
| Identity (S15) | New section: `deviceId` conflict detection, data retention policy |
