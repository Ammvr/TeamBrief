# Playbook — TeamBrief Task Decomposition & Reference

> Load this when planning work or assigning tasks. Core rules live in `orchestrator.md` (always loaded).

---

## Project Summary

TeamBrief converts messy daily updates into structured standups (Done / Next / Blockers), with an intelligent Blocker Board and Markdown export. PRD is the authoritative specification.

---

## Configurable Thresholds (`src/shared/config.ts`)

```typescript
export const CONFIG = {
  // AI
  AI_MODEL: "gpt-4o-mini",
  EMBEDDING_MODEL: "text-embedding-3-small",  // Use default dimensions

  // Clustering
  SIMILARITY_THRESHOLD: 0.85,  // Initial default for short text. Tune on demo data. Range: 0.82–0.90

  // Display caps (per person in Today Standup view)
  MAX_DONE_DISPLAY: 3,
  MAX_NEXT_DISPLAY: 3,
  MAX_BLOCKERS_DISPLAY: 2,

  // Quality checks
  MIN_INPUT_CHARS: 8,
  MIN_INPUT_WORDS: 2,
  MAX_ITEM_LENGTH: 160,
  MULTI_TASK_HARD_THRESHOLD: 3,  // 3+ actions = hard fail
  MULTI_TASK_SOFT_THRESHOLD: 2,  // 2 actions = soft fail

  // AI retry
  AI_RETRY_BACKOFF_MIN_MS: 800,
  AI_RETRY_BACKOFF_MAX_MS: 1200,

  // Blocker Board
  RESURFACED_WINDOW_DAYS: 7,
  BLOCKER_HIGHLIGHTS_COUNT: 3,

  // Export
  BLOCKER_CATEGORIES: ["Access", "Decision", "Dependency", "Technical", "External", "Capacity"] as const,
} as const;
```

All sub-agents: "Import thresholds from `src/shared/config.ts`. Do not define local constants for any value that exists in CONFIG."

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Files/folders | kebab-case | `blocker-board.tsx`, `quality-check.ts` |
| React components | PascalCase export, kebab-case file | `blocker-board.tsx` → `BlockerBoard` |
| Functions/variables | camelCase | `qualityCheck()`, `clusteringStatus` |
| Types/interfaces | PascalCase | `UpdateSubmission`, `BlockerCluster` |
| API routes | kebab-case paths | `/api/blocker-clusters` |
| Enums | PascalCase name, values matching PRD | `ExtractionStatus.SUCCESS` |

### Standard Error Response
```typescript
{ error: string; code?: string }
```
HTTP status: 400 (validation), 404 (not found), 500 (server error).

### Null Handling
Nullable in data model = handled explicitly in API and UI. Never assume non-null unless Prisma schema enforces it.

---

## Auto-Cut List (DEFERRED — do NOT build unless re-scoped)

- deviceId conflict detection/warning UI (generation is kept, conflict detection is cut)
- Quality score computation (rolling average)
- By-Person export view (Team Summary only for MVP)
- Resurfaced badge tooltip with full resolution history (basic badge is P2, tooltip is cut)

---

## Task Decomposition

---

### PHASE 1 — Deployable E2E Demo

#### Layer 0 — Foundation (blocks everything)

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-00 | MiniMax M2.5 | `P1-CRITICAL` | Project scaffolding: Next.js App Router, Prisma init with `DATABASE_URL`, folder structure (`src/shared/`, `src/components/`, `src/app/api/`, `src/lib/`), tsconfig strict, environment setup, install dependencies (zod, openai, @prisma/client). Enable pgvector extension in Prisma schema for future use. |
| TASK-01 | GLM-5 | `P1-CRITICAL` | Canonical contract files: `src/shared/contracts.ts` (all TypeScript interfaces/types/enums matching PRD data model — Project, Update, BlockerItem, BlockerCluster, all enums, all API request/response shapes), `src/shared/schemas.ts` (Zod schemas for API input validation and AI output validation), `src/shared/config.ts` (all configurable constants as specified in this playbook). |

**Run TASK-00 and TASK-01 in parallel.**

---

#### Layer 1 — Data & Core Logic (after Layer 0)

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-02 | MiniMax M2.5 | `P1-CRITICAL` | Prisma schema + migration matching PRD data model. All fields, relations, enums, indexes. Vector column for future embedding storage (pgvector). Import types from contracts.ts for reference. |
| TASK-03 | MiniMax M2.5 | `P1-CRITICAL` | API routes: Projects CRUD (`/api/projects`), Updates submission (`/api/updates` — receives raw text, calls AI-1 extraction function, runs quality checks, returns extracted items + flags), Today Standup query (`/api/standup?projectId=X&date=Y`). **Micro-dependency**: builds route skeletons against TASK-01 contracts. Stubs AI-1 and quality check internals until TASK-04 and TASK-05 finalize, then wires them in. |
| TASK-04 | GLM-5 | `P1-CRITICAL` | AI-1 extraction function: OpenAI gpt-4o-mini call with Structured Outputs. Strict JSON schema `{ done[], next[], blockers[] }` with max caps from CONFIG. reviewReason flagging. In-progress→Next policy. "Do not add new facts" system prompt. Returns validated output using Zod schema from schemas.ts. Exported as standalone async function that TASK-03 imports. |
| TASK-05 | GLM-5 | `P1-CRITICAL` | Quality check engine: server-side `qualityCheck()` function. Two-tier severity (hard fail / soft fail). Vague detection, multi-task heuristic (conjunction counting from CONFIG), no-action detection, too-long detection. Returns flagged items with severity and reason. Exported as standalone function that TASK-03 imports. |
| TASK-07 | GLM-5 | `P1-CRITICAL` | Zod validation for API route inputs (submit update request body, standup query params, project creation). Integrated into schemas.ts. Runs in route handlers before any processing. |

**Run all Layer 1 tasks in parallel. TASK-03 stubs internals until TASK-04/05/07 deliver.**

---

#### Layer 2 — Frontend for E2E (after Layer 0 contracts + Layer 1 API contracts defined)

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-10 | Kimi K2.5 | `P1-CRITICAL` | Submit Update page: project selector (create/select), name input (localStorage persistence), free-form text area, client-side input gate (CONFIG thresholds — min chars, min words, letters required, stopword rejection with friendly error message from PRD), submit calls `POST /api/updates`. |
| TASK-11 | Kimi K2.5 | `P1-CRITICAL` | Confirmation screen: extracted Done/Next/Blockers from API response. Inline reviewReason highlighting. Hard-fail blocks submit (must edit to clear). Soft-fail dismissible. Nudges for empty next/blockers (Add Next / Add Blocker / Confirm None). Edit capability. Final submit stores confirmed data via `POST /api/updates/confirm`. **Must also work with manually entered items** (Demo Reliability). |
| TASK-12 | Kimi K2.5 | `P1-CRITICAL` | Today Standup view: fetches `/api/standup?projectId=X&date=Y`. Aggregated Done/Next/Blockers per project+date. Per-person display caps from CONFIG. "View Blockers" link (placeholder until Phase 2). |
| TASK-14 | Kimi K2.5 | `P1-CRITICAL` | Export modal (basic): from Today Standup. Format selector (Slack/Notion). Markdown — Notion: `## Done` + `- item`, Slack: `*Done*` + `• item`. Team Summary only. One-click copy-to-clipboard. |
| TASK-15 | Kimi K2.5 | `P1-CRITICAL` | Client utilities: deviceId UUID generation + localStorage persistence (sent with every submission). Loading states for all API calls. Error boundaries for all pages. Navigation layout (project selector in header, routing between Submit / Today / Blocker Board). |

**Run all Layer 2 tasks in parallel. Build against contracts, not backend implementation.**

---

#### Layer 3 — Phase 1 Integration + Deploy

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-16A | GLM-5 (Integration Reviewer) | `P1-CRITICAL` | Integration Review across all Phase 1 tasks (Layers 0–2). Mismatch list only. |
| TASK-16B | (Fix agents) | `P1-CRITICAL` | Apply fixes per Retry Protocol. |
| TASK-16C | MiniMax M2.5 | `P1-CRITICAL` | Vercel deployment: build passes, env vars configured, Prisma client generated, all routes functional. Smoke test E2E flow. |

**TASK-16A → TASK-16B → TASK-16C. Contracts freeze after TASK-16A passes.**

**🚨 PHASE 1 GATE: Do not proceed to Phase 2 until E2E is deployed on Vercel.**

---

### PHASE 2 — Blocker Intelligence + Board

#### Layer 4 — Background Pipeline + Board UI

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-06 | GLM-5 | `P2-BLOCKER-BOARD` | AI-3 rewrite suggestions: conditional gpt-4o-mini call for hard-fail items, 2–3 suggestions each. Zod-validated. Wires into confirmation screen (hard-fails now show AI suggestions alongside manual editing). |
| TASK-08 | GLM-5 | `P2-BLOCKER-BOARD` | AI-2 blocker categorization: async post-submit. Batch per update, assigns one of 6 categories from CONFIG.BLOCKER_CATEGORIES. Does NOT block user. Stores on BlockerItem. |
| TASK-09 | GLM-5 | `P2-BLOCKER-BOARD` | Embedding + clustering pipeline: async post-submit. Embedding via text-embedding-3-small (default dimensions). Cosine similarity vs cluster centroids (same project + same category). Threshold from CONFIG (0.85 initial, tune on demo data). Centroid running average. New cluster creation. Resurfacing logic (RESOLVED→OPEN, increment resurfacedCount, set lastResurfacedAt). clusteringStatus tracking. |
| TASK-03B | MiniMax M2.5 | `P2-BLOCKER-BOARD` | API routes: `/api/blocker-clusters?projectId=X` (clusters with aging + blast radius), `PATCH /api/blocker-clusters/[id]/status` (toggle Open/Resolved), `POST /api/pipeline/run` with body `{ updateId }` (triggers async categorization + embedding + clustering). |
| TASK-13 | Kimi K2.5 | `P2-BLOCKER-BOARD` | Blocker Board page: fetches `/api/blocker-clusters`. Cluster cards: representativeText, category badge, aging (days since firstSeenAt), blast radius (uniqueMemberCount), status toggle, Resurfaced badge (resurfacedCount > 0, lastResurfacedAt within CONFIG.RESURFACED_WINDOW_DAYS). "Updating clusters…" spinner for PENDING state. |

**Run all Layer 4 tasks in parallel. Must not break Phase 1 flow.**

---

#### Layer 5 — Phase 2 Integration

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-16D | GLM-5 (Integration Reviewer) | `P2-BLOCKER-BOARD` | Integration Review on Phase 2 tasks. Mismatch list only. |
| TASK-16E | (Fix agents) | `P2-BLOCKER-BOARD` | Apply fixes per Retry Protocol. |

---

### PHASE 3 — Hardening + Demo Reliability

#### Layer 6 — Polish

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-17 | MiniMax M2.5 | `P3-HARDENING` | Error handling: AI-1 retry with jitter backoff (CONFIG), malformed output repair (re-prompt), manual fallback screen (empty fields + raw text, extractionStatus=FAILED), async pipeline failure (clusteringStatus=FAILED, uncategorized on Board). |
| TASK-18 | Kimi K2.5 | `P3-HARDENING` | Demo seed button: "Seed demo updates" inserts pre-extracted updates + pre-creates blocker clusters. No AI calls. Demo insurance via `POST /api/demo/seed`. |
| TASK-19 | Kimi K2.5 | `P3-HARDENING` | Export enhancement: Blocker Highlights footer (top CONFIG.BLOCKER_HIGHLIGHTS_COUNT clusters with category, age, blast radius). |

**Run Layer 6 tasks in parallel.**

---

### DEFERRED

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-20 | Kimi K2.5 | `DEFERRED` | Resurfaced badge tooltip with full resolution history. |
| TASK-21 | Kimi K2.5 | `DEFERRED` | deviceId conflict warning UI. |
| TASK-22 | GLM-5 | `DEFERRED` | Quality score computation (rolling average). |
| TASK-23 | Kimi K2.5 | `DEFERRED` | By-Person export view toggle. |

---

## Parallelism Map

```
PHASE 1 — Deployable E2E
─────────────────────────
Layer 0:  [TASK-00] + [TASK-01]
              │
Layer 1:  [TASK-02] + [TASK-03*] + [TASK-04] + [TASK-05] + [TASK-07]
              │        (* stubs until 04/05/07)
Layer 2:  [TASK-10] + [TASK-11] + [TASK-12] + [TASK-14] + [TASK-15]
              │        (starts after Layer 0 contracts)
Layer 3:  [TASK-16A] → [TASK-16B] → [TASK-16C deploy]
              │
      🚨 GATE — deployed on Vercel, contracts frozen
              │
PHASE 2 — Blocker Intelligence
─────────────────────────────
Layer 4:  [TASK-06] + [TASK-08] + [TASK-09] + [TASK-03B] + [TASK-13]
              │
Layer 5:  [TASK-16D] → [TASK-16E]
              │
PHASE 3 — Hardening
─────────────────────────────
Layer 6:  [TASK-17] + [TASK-18] + [TASK-19]
```

---

## Pre-Planned Endpoints (complete list)

These are all API endpoints for the entire project. No additions after Phase 1 deploys (except these pre-planned ones).

### Phase 1
- `POST /api/projects` — create project
- `GET /api/projects` — list projects
- `POST /api/updates` — submit raw text, returns AI extraction + quality flags
- `POST /api/updates/confirm` — store confirmed/edited update
- `GET /api/standup?projectId=X&date=Y` — today standup aggregation

### Phase 2 (pre-planned)
- `GET /api/blocker-clusters?projectId=X` — clusters with aging + blast radius
- `PATCH /api/blocker-clusters/[id]/status` — toggle Open/Resolved
- `POST /api/pipeline/run` with body `{ updateId }` — trigger async categorization + clustering

### Phase 3 (pre-planned)
- `POST /api/demo/seed` — insert pre-extracted demo data
