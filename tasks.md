# Tasks & Milestones — TeamBrief Hackathon Build

> Consistent with: `orchestrator.md` (v3.1), `playbook.md`, `TeamBrief_PRD.md` (v2.1)
>
> Reference: All tasks follow the phase gates, priority tags, agent assignments, and contract architecture defined in the orchestrator and playbook.

---

## Milestones

| Milestone | Gate | Definition of Done |
|-----------|------|-------------------|
| **M0 — Foundation** | Layer 0 complete | Project scaffolded, contracts/schemas/config canonical files created, folder structure in place, dependencies installed. |
| **M1 — Core Logic** | Layer 1 complete | Prisma schema migrated, API routes functional (stubs wired to real AI-1 + quality check), Zod validation active on all inputs. |
| **M2 — Frontend E2E** | Layer 2 complete | All Phase 1 UI pages render and communicate with API. Submit → Confirm → Store → Today View → Export flow works end-to-end in local dev. |
| **M3 — Phase 1 Deployed** | 🚨 Phase 1 Gate | Integration review passes. E2E flow deployed on Vercel. Manual entry fallback works. This is the minimum viable demo. **Nothing else proceeds until M3 is met.** |
| **M4 — Blocker Intelligence** | Layer 4 complete | AI-2 categorization, embedding + clustering pipeline, and Blocker Board UI functional. Async pipeline triggered via `/api/pipeline/run`. |
| **M5 — Phase 2 Integrated** | Layer 5 complete | Phase 2 integration review passes. Blocker Board shows clustered, categorized blockers with aging and blast radius. AI-3 suggestions wired into confirmation screen. |
| **M6 — Demo Ready** | Layer 6 complete | Error handling hardened, demo seed button functional, export has Blocker Highlights footer. Demo survives OpenAI outage via manual entry + seed data. |

---

## Tasks

### PHASE 1 — Deployable E2E Demo

---

#### Layer 0 — Foundation → Milestone M0

##### TASK-00 | MiniMax M2.5 | `P1-CRITICAL`
**Objective**: Scaffold the Next.js project with all infrastructure ready for development.

**Deliverables**:
- Next.js App Router project initialized
- Prisma initialized with `DATABASE_URL` connection to Supabase Postgres
- pgvector extension enabled in Prisma schema (for future embedding use)
- Folder structure: `src/shared/`, `src/components/`, `src/app/api/`, `src/lib/`
- `tsconfig.json` with strict mode enabled
- Dependencies installed: `zod`, `openai`, `@prisma/client`, `@prisma/extension-accelerate` (if needed)
- `.env.example` with `DATABASE_URL`, `OPENAI_API_KEY` (server-side only)
- `.gitignore` properly configured

**Acceptance Criteria**: `npm run dev` starts without errors. Prisma connects to Supabase. Folder structure matches spec.

---

##### TASK-01 | GLM-5 | `P1-CRITICAL`
**Objective**: Create the three canonical contract files that serve as the single source of truth.

**Deliverables**:
- `src/shared/contracts.ts` — All TypeScript interfaces, types, and enums:
  - Data model types: `Project`, `Update`, `BlockerItem`, `BlockerCluster`
  - Enums: `ExtractionStatus` (SUCCESS | FAILED | MANUAL), `ClusteringStatus` (PENDING | COMPLETED | FAILED), `BlockerCategory` (Access | Decision | Dependency | Technical | External | Capacity), `ClusterStatus` (OPEN | RESOLVED), `QualitySeverity` (HARD_FAIL | SOFT_FAIL), `QualityFlag` (vague_severe | multi_task_severe | no_action | too_long | vague_soft | multi_task_soft)
  - API request/response shapes: `CreateProjectRequest`, `CreateProjectResponse`, `SubmitUpdateRequest`, `SubmitUpdateResponse` (includes extracted items + quality flags), `ConfirmUpdateRequest`, `StandupQueryParams`, `StandupResponse`, `BlockerClustersResponse`, `PipelineRunRequest`
  - Extracted item types: `ExtractedItem` (text, reviewReason?, qualityFlags?), `ExtractionResult` (done[], next[], blockers[])
  - Quality check types: `QualityCheckResult`, `QualityFlag` with severity
- `src/shared/schemas.ts` — Zod schemas:
  - `createProjectSchema`, `submitUpdateSchema`, `confirmUpdateSchema`, `standupQuerySchema`
  - `aiExtractionOutputSchema` (strict: done[max 3], next[max 3], blockers[max 2], item max 160 chars)
  - `qualityCheckResultSchema`
- `src/shared/config.ts` — All configurable constants as defined in the playbook CONFIG block

**PRD References**: Section 10 (Data Model), Section 7 (AI Usage), Section 8 (Functional Requirements), Section 9 (Quality/Normalization)

**Acceptance Criteria**: All types compile with `tsc --noEmit`. Zod schemas match TypeScript types. CONFIG contains all thresholds from playbook. No `any` types.

---

#### Layer 1 — Data & Core Logic → Milestone M1

##### TASK-02 | MiniMax M2.5 | `P1-CRITICAL`
**Objective**: Create Prisma schema and run migration matching the PRD data model.

**Deliverables**:
- `prisma/schema.prisma` with models: `Project`, `Update`, `BlockerItem`, `BlockerCluster`
- All fields, types, relations, and enums matching PRD Section 10
- Vector column on `BlockerItem` (embedding) and `BlockerCluster` (centroidEmbedding) using pgvector
- Indexes: `Update` on (projectId, date), `BlockerItem` on (projectId, clusterId), `BlockerCluster` on (projectId, category)
- Migration runs successfully against Supabase

**Consumes**: `src/shared/contracts.ts` (types for reference)

**PRD References**: Section 10 (Data Model)

**Acceptance Criteria**: `npx prisma migrate dev` succeeds. `npx prisma generate` produces client. All models match PRD field specs.

---

##### TASK-03 | MiniMax M2.5 | `P1-CRITICAL`
**Objective**: Build Phase 1 API route handlers.

**Deliverables**:
- `src/app/api/projects/route.ts` — `POST` (create project), `GET` (list projects)
- `src/app/api/updates/route.ts` — `POST` (receives raw text + memberName + deviceId + projectId, calls AI-1 extraction, runs quality checks, returns extracted items + flags + reviewReasons)
- `src/app/api/updates/confirm/route.ts` — `POST` (receives confirmed/edited items, stores Update record with doneJson/nextJson/blockersJson, creates BlockerItem records for any blockers, sets extractionStatus)
- `src/app/api/standup/route.ts` — `GET` (query by projectId + date, returns aggregated standup with per-person display caps from CONFIG)
- All routes validate input using Zod schemas from `schemas.ts`
- All routes return standard error shape: `{ error: string; code?: string }`
- Status code policy:
  - `400` — Zod validation failure (malformed input)
  - `404` — Resource not found (project, update)
  - `500` — Server error with `code` field: `"AI_EXTRACTION_FAILED"`, `"AI_SUGGESTION_FAILED"`, `"DB_ERROR"`, `"PIPELINE_FAILED"`
  - No 401/403 (no auth in MVP)

**Micro-dependency**: Builds route skeletons against TASK-01 contracts. Stubs AI-1 extraction (TASK-04) and quality check (TASK-05) internals until those tasks deliver, then wires them in.

**Consumes**: `contracts.ts`, `schemas.ts`, `config.ts`, Prisma client (TASK-02)
**Produces**: Working API endpoints for frontend consumption

**PRD References**: Section 5 (User Flows), Section 8 (FR-1 through FR-6), Section 12.3 (Input Gate)

**Acceptance Criteria**: All routes respond correctly via curl/Postman. Zod rejects malformed input with 400. Standup query returns correctly aggregated and capped data.

---

##### TASK-04 | GLM-5 | `P1-CRITICAL`
**Objective**: Build the AI-1 extraction function.

**Deliverables**:
- `src/lib/ai/extract.ts` — exports `extractUpdate(rawText: string): Promise<ExtractionResult>`
- Calls OpenAI gpt-4o-mini with Structured Outputs (strict JSON mode)
- System prompt enforces: "Do not add new facts. Only rewrite/split what the user provided."
- In-progress statements → classified as Next, not Done (PRD Section 7.1 policy)
- Output capped: done max 3, next max 3, blockers max 2 (from CONFIG)
- Adds `reviewReason: string` on items using these deterministic triggers:
  - **Reclassification**: any in-progress statement moved from Done to Next (e.g., "Marked as Next — sounds in-progress, not completed")
  - **Hedging language**: item contains hedging words (`maybe`, `might`, `some`, `stuff`, `things`, `working on`, `trying to`)
  - **Ambiguous category**: item could plausibly belong to a different category (e.g., a blocker that reads like a next-step)
  - reviewReason is a short human-readable string explaining why the item is flagged (e.g., "Contains hedging language — verify this is complete")
- Output validated against Zod schema from `schemas.ts` before returning
- Uses `CONFIG.AI_MODEL` — no hardcoded model string

**Consumes**: `contracts.ts` (ExtractionResult type), `schemas.ts` (aiExtractionOutputSchema), `config.ts`

**PRD References**: Section 7.1 (AI-1)

**Acceptance Criteria**: Given sample messy updates, returns correctly structured JSON. In-progress items appear in `next[]` with reviewReason explaining the reclassification. No invented facts. Hedging-language items get reviewReason. Output passes Zod validation.

---

##### TASK-05 | GLM-5 | `P1-CRITICAL`
**Objective**: Build the quality check engine.

**Deliverables**:
- `src/lib/quality/check.ts` — exports `qualityCheck(items: ExtractedItem[], category: 'done' | 'next' | 'blockers'): QualityCheckResult[]`
- Implements two-tier severity (PRD Section 9.1):
  - **Hard fail**: `vague_severe` (no specifics), `multi_task_severe` (3+ actions via 2+ conjunctions), `no_action` (no verb), `too_long` (paragraph-length)
  - **Soft fail**: slightly vague, 2 actions, missing next/blockers
- Multi-task heuristic (PRD Section 9.2): counts conjunctions (`and`, `then`, `also`, `plus`, `&`) and separators; 3+ actions = hard, 2 = soft
- All thresholds from CONFIG (MULTI_TASK_HARD_THRESHOLD, MULTI_TASK_SOFT_THRESHOLD, MAX_ITEM_LENGTH)
- Returns per-item flags with severity and human-readable reason

**Consumes**: `contracts.ts` (QualityCheckResult, QualitySeverity, QualityFlag types), `config.ts`

**PRD References**: Section 9 (Quality/Normalization)

**Acceptance Criteria**: "worked on backend" → hard fail (vague_severe). "refactored auth and added tests" → soft fail. "Completed login page redesign with responsive breakpoints" → pass. 3+ conjunction item → hard fail.

---

##### TASK-07 | GLM-5 | `P1-CRITICAL`
**Objective**: Ensure all API route Zod schemas are complete and integrated.

**Deliverables**:
- Verify/complete all schemas in `src/shared/schemas.ts` for:
  - `POST /api/projects` request body
  - `POST /api/updates` request body (rawText, memberName, deviceId, projectId)
  - `POST /api/updates/confirm` request body (confirmed items arrays)
  - `GET /api/standup` query params (projectId, date)
- Schemas enforce: required fields, string length limits, UUID format, date format
- Schemas strip empty strings and duplicate items from confirmed arrays

**Consumes**: `contracts.ts` (API shapes)
**Produces**: Finalized `schemas.ts` used by all route handlers

**PRD References**: Section 12.2 (Malformed AI Output — Zod validation)

**Acceptance Criteria**: All schemas reject malformed input with descriptive errors. All schemas accept valid input. No route handler does ad-hoc validation — all use schemas.

---

#### Layer 2 — Frontend for E2E → Milestone M2

##### TASK-10 | Kimi K2.5 | `P1-CRITICAL`
**Objective**: Build the Submit Update page.

**Deliverables**:
- `src/app/page.tsx` or `src/app/submit/page.tsx` — main submit page
- `src/components/project-selector.tsx` — create/select project dropdown (calls `GET/POST /api/projects`)
- `src/components/update-form.tsx` — name input (persists to localStorage), free-form text area, submit button
- Client-side input gate before API call: rejects < CONFIG.MIN_INPUT_CHARS, < CONFIG.MIN_INPUT_WORDS, no letters, only stopwords
- Friendly error message from PRD Section 12.3: "Your update is too short to extract..."
- On submit: calls `POST /api/updates`, navigates to confirmation screen with response data
- Uses `"use client"` directive where needed

**Consumes**: `contracts.ts` (SubmitUpdateRequest/Response), `config.ts` (input gate thresholds)

**PRD References**: Section 5 Flow A (steps 1–4), Section 6.2, Section 12.3

**Acceptance Criteria**: Project creation works. Name persists across refreshes. Input gate rejects "hi" and accepts 2+ sentence updates. Submission calls API and navigates to confirmation.

---

##### TASK-11 | Kimi K2.5 | `P1-CRITICAL`
**Objective**: Build the Confirmation screen.

**Deliverables**:
- `src/app/confirm/page.tsx` or `src/components/confirmation-screen.tsx`
- Displays extracted Done / Next / Blockers from API response
- Items with `reviewReason` are visually highlighted (distinct styling)
- Hard-fail items: blocked, user must edit to clear the flag before submit is enabled
- Soft-fail items: warning shown, dismissible, submit not blocked
- Empty `next[]` or `blockers[]`: nudge with one-tap actions (Add Next / Add Blocker / Confirm None)
- Each item is editable inline
- Final submit calls `POST /api/updates/confirm`
- After confirm: navigates to Today Standup view
- **Demo Reliability**: screen MUST work with manually entered items (empty extraction + raw text visible). User can fill Done/Next/Blockers from scratch if AI-1 fails.
- After confirm succeeds and blockers exist: fires `POST /api/pipeline/run` with body `{ updateId }` (async, non-blocking). **If this returns 404 (endpoint not yet built in Phase 1), silently ignore it. Do NOT show any error toast, notification, or console warning to the user.** This call becomes functional in Phase 2.

**Consumes**: `contracts.ts` (SubmitUpdateResponse, ConfirmUpdateRequest), `config.ts`

**PRD References**: Section 5 Flow A (steps 5–9), Section 6.3, Section 9.1

**Acceptance Criteria**: Hard-fail items block submit until edited. Soft-fail items are dismissible. Empty next shows nudge. Manual entry mode works. Confirm stores data and redirects.

---

##### TASK-12 | Kimi K2.5 | `P1-CRITICAL`
**Objective**: Build the Today Standup view.

**Deliverables**:
- `src/app/standup/page.tsx`
- Fetches `GET /api/standup?projectId=X&date=Y` (defaults to today)
- Displays aggregated Done / Next / Blockers grouped by member
- Per-person display caps applied: Done ≤ CONFIG.MAX_DONE_DISPLAY, Next ≤ CONFIG.MAX_NEXT_DISPLAY, Blockers ≤ CONFIG.MAX_BLOCKERS_DISPLAY
- "View Blockers" link (navigates to `/blocker-board` — placeholder page until Phase 2)
- "Export" button opens export modal (TASK-14)
- Handles empty state (no updates yet for today)

**Consumes**: `contracts.ts` (StandupResponse), `config.ts`

**PRD References**: Section 5 Flow B, Section 6.4, FR-5

**Acceptance Criteria**: Shows aggregated updates per person with correct caps. Empty state handled. View Blockers link exists. Export button triggers modal.

---

##### TASK-14 | Kimi K2.5 | `P1-CRITICAL`
**Objective**: Build the basic Markdown export modal.

**Deliverables**:
- `src/components/export-modal.tsx`
- Format selector: Slack or Notion
- Markdown generation:
  - Notion: `## Done` + `- item` (real Markdown)
  - Slack: `*Done*` + `• item` (Slack formatting)
- Team Summary view only (By-Person is DEFERRED)
- One-click copy-to-clipboard button with feedback ("Copied!")
- Modal triggered from Today Standup view

**Consumes**: `contracts.ts` (StandupResponse data), `config.ts`

**PRD References**: Section 5 Flow C, Section 6.6, FR-12

**Acceptance Criteria**: Both formats generate correct Markdown. Copy button works. Modal opens/closes cleanly.

---

##### TASK-15 | Kimi K2.5 | `P1-CRITICAL`
**Objective**: Build client-side utilities and app shell.

**Deliverables**:
- `src/lib/device-id.ts` — deviceId UUID generation on first visit, localStorage persistence, exported getter function
- `src/components/layout/app-layout.tsx` — navigation header with project selector, page routing (Submit / Today Standup / Blocker Board)
- `src/components/ui/loading-spinner.tsx` — reusable loading indicator
- `src/components/ui/error-boundary.tsx` — error boundary wrapper for all pages
- Loading states on all API-calling components (show spinner during fetch)
- deviceId included in all submission API calls automatically

**Consumes**: `contracts.ts`

**PRD References**: Section 15.1 (deviceId generation only — conflict detection is DEFERRED), FR-13

**Acceptance Criteria**: deviceId persists across page refreshes. Navigation works between all pages. Loading states visible during API calls. Error boundary catches render errors.

---

#### Layer 3 — Integration + Deploy → Milestone M3

**Gate: Phase 1 is not considered complete until TASK-16A (Integration Review) is done by GLM-5 and marked PASS. This is blocking — not advisory. Contracts do not freeze and Phase 2 does not begin until PASS is achieved.**

**Emergency exception:** PASS-with-warnings is allowed only with explicit user approval. GLM-5 lists the warnings, you (user) consciously accept the debt.

##### TASK-16A | GLM-5 (Integration Reviewer) | `P1-CRITICAL`
**Objective**: Run integration review across all Phase 1 tasks. **This is a blocking gate.**

**Scope**: All files from Layers 0–2. Check every consumer-producer interface against `contracts.ts`.

**PASS Checklist (all must be true):**
- All API routes validate inputs and outputs with Zod schemas from `schemas.ts`
- All response shapes match types defined in `contracts.ts`
- No duplicated local types that should be in `contracts.ts`
- No hardcoded thresholds — all values come from `config.ts`
- UI `fetch()` calls match the API endpoint paths, methods, and request/response shapes
- Async pipeline call failure (404 in Phase 1) is silently ignored — no error toast or user-visible warning
- Status code policy is consistent: 400 (Zod), 404 (not found), 500 (server error with `code` field)
- deviceId is generated, persisted, and sent with submissions
- Manual entry fallback works on confirmation screen (Demo Reliability)

**Output**: `PASS`, `PASS-WITH-WARNINGS` (requires user approval to proceed), or `FAIL` with mismatch list (file paths, expected vs actual, MECHANICAL/STRUCTURAL severity).

**Triggers on PASS**: Contract Freeze. Phase 2 may begin.

---

##### TASK-16B | (Assigned per mismatch) | `P1-CRITICAL`
**Objective**: Fix all mismatches found by TASK-16A.

**Protocol**:
- MECHANICAL: one precise fix instruction to original agent. One retry.
- STRUCTURAL: escalate to user immediately.

---

##### TASK-16C | MiniMax M2.5 | `P1-CRITICAL`
**Objective**: Deploy Phase 1 to Vercel.

**Deliverables**:
- Vercel project configured
- Environment variables set: `DATABASE_URL`, `OPENAI_API_KEY`
- `prisma generate` runs in build step
- Build passes without errors
- Smoke test: submit update → confirm → view standup → export markdown

**Acceptance Criteria**: E2E flow works on deployed Vercel URL. **Report: "🚀 PHASE 1 DEPLOYED — [URL]. Proceeding to Phase 2."**

---

### 🚨 PHASE 1 GATE
**Contracts frozen. No new endpoints except pre-planned. Phase 2 begins.**

---

### PHASE 2 — Blocker Intelligence + Board

---

#### Layer 4 — Background Pipeline + Board UI → Milestone M4

##### TASK-06 | GLM-5 | `P2-BLOCKER-BOARD`
**Objective**: Build AI-3 rewrite suggestions function and wire into confirmation flow.

**Deliverables**:
- `src/lib/ai/suggest.ts` — exports `getRewriteSuggestions(item: ExtractedItem, flag: QualityFlag): Promise<string[]>`
- Calls gpt-4o-mini, returns 2–3 rewrite suggestions per flagged item
- Output validated via Zod
- Wire into `POST /api/updates` response: when hard-fail flags exist, include suggestions alongside flags
- Update confirmation screen to display suggestions for hard-fail items (user picks one or edits manually)

**Consumes**: `contracts.ts`, `schemas.ts`, `config.ts`

**PRD References**: Section 7.3 (AI-3)

**Acceptance Criteria**: Hard-fail items now show rewrite suggestions. User can pick a suggestion or still edit manually. Soft-fail items unchanged.

---

##### TASK-08 | GLM-5 | `P2-BLOCKER-BOARD`
**Objective**: Build AI-2 blocker categorization function.

**Deliverables**:
- `src/lib/ai/categorize.ts` — exports `categorizeBlockers(blockers: BlockerItem[]): Promise<BlockerCategory[]>`
- Calls gpt-4o-mini, assigns one of CONFIG.BLOCKER_CATEGORIES per blocker
- Batched per update (all blockers from one submission in one call)
- Output validated via Zod
- Updates BlockerItem records with category

**Consumes**: `contracts.ts` (BlockerCategory enum), `config.ts`

**PRD References**: Section 7.2 (AI-2), FR-7

**Acceptance Criteria**: Given blocker texts, assigns correct categories. "Waiting for AWS access" → Access. "Need PM decision on scope" → Decision.

---

##### TASK-09 | GLM-5 | `P2-BLOCKER-BOARD`
**Objective**: Build embedding generation and clustering pipeline.

**Deliverables**:
- `src/lib/clustering/embed.ts` — exports `generateEmbedding(text: string): Promise<number[]>`
  - Calls text-embedding-3-small (default dimensions)
- `src/lib/clustering/cluster.ts` — exports `clusterBlocker(blockerItem: BlockerItem): Promise<void>`
  - Queries existing clusters (same project + same category)
  - Computes cosine similarity between new embedding and each cluster centroid
  - If similarity > CONFIG.SIMILARITY_THRESHOLD: assign to best match, update centroid using running average formula: `newCentroid[i] = (oldCentroid[i] * oldCount + newVec[i]) / (oldCount + 1)`, update cluster metadata (count, lastSeenAt, uniqueMemberCount)
  - Note: text-embedding-3-small returns normalized vectors; no manual normalization needed before cosine similarity
  - If no match: create new cluster with this blocker as seed (embedding becomes initial centroid)
  - If matched cluster has status=RESOLVED: set OPEN, increment resurfacedCount, set lastResurfacedAt
  - Updates Update.clusteringStatus to COMPLETED (or FAILED on error)
- `src/lib/clustering/pipeline.ts` — exports `runPipeline(updateId: string): Promise<void>`
  - Orchestrates: categorize blockers → generate embeddings → cluster each blocker
  - Called by `/api/pipeline/run`

**Consumes**: `contracts.ts`, `config.ts`, Prisma client

**PRD References**: Section 7.4, Section 11 (Clustering Approach), FR-8 through FR-11

**Acceptance Criteria**: Similar blocker texts cluster together. Different topics create separate clusters. Resurfacing logic triggers correctly. clusteringStatus updates on Update record.

---

##### TASK-03B | MiniMax M2.5 | `P2-BLOCKER-BOARD`
**Objective**: Build Phase 2 API routes.

**Deliverables**:
- `src/app/api/blocker-clusters/route.ts` — `GET` (query by projectId, returns clusters with computed aging in days and blast radius)
- `src/app/api/blocker-clusters/[id]/status/route.ts` — `PATCH` (toggle Open/Resolved, stores resolvedAt)
- `src/app/api/pipeline/run/route.ts` — `POST` (receives updateId, calls `runPipeline()`, returns 200 on success)
- All routes use Zod validation and standard error responses

**Consumes**: `contracts.ts`, `schemas.ts`, clustering pipeline (TASK-09)

**PRD References**: FR-7 through FR-11

**Acceptance Criteria**: Blocker clusters endpoint returns correctly computed aging and blast radius. Status toggle persists. Pipeline run triggers categorization + clustering.

---

##### TASK-13 | Kimi K2.5 | `P2-BLOCKER-BOARD`
**Objective**: Build the Blocker Board page.

**Deliverables**:
- `src/app/blocker-board/page.tsx`
- Fetches `GET /api/blocker-clusters?projectId=X`
- Cluster cards displaying:
  - `representativeText` (cluster label)
  - Category badge (color-coded)
  - Aging: "X days" (days since firstSeenAt)
  - Blast radius: "X members affected" (uniqueMemberCount)
  - Status toggle button (Open ↔ Resolved via `PATCH /api/blocker-clusters/[id]/status`)
  - Resurfaced badge (visible when resurfacedCount > 0 AND lastResurfacedAt within CONFIG.RESURFACED_WINDOW_DAYS)
- "Updating clusters…" spinner shown when any recent update has clusteringStatus = PENDING
- Empty state when no clusters exist
- Sorted by: Open first, then by aging (longest first)

**Consumes**: `contracts.ts` (BlockerClustersResponse), `config.ts`

**PRD References**: Section 6.5, FR-8 through FR-11

**Acceptance Criteria**: Clusters display with correct metadata. Status toggle works. Resurfaced badge appears correctly. Spinner shows during async processing.

---

#### Layer 5 — Phase 2 Integration → Milestone M5

**Gate: Phase 2 is not considered complete until TASK-16D (Integration Review) is done by GLM-5 and marked PASS. This is blocking.**

##### TASK-16D | GLM-5 (Integration Reviewer) | `P2-BLOCKER-BOARD`
**Objective**: Integration review across Phase 2 tasks. **This is a blocking gate.**

**Scope**: All Phase 2 files. Check against frozen contracts.

**PASS Checklist (all must be true):**
- All Phase 2 routes validate inputs/outputs with Zod
- Blocker Board UI fetches and renders data matching `contracts.ts` response shapes
- Async pipeline (categorize → embed → cluster) executes without breaking Phase 1 E2E flow
- Cluster status toggle persists correctly
- No new types outside `contracts.ts` (contracts are frozen)
- All thresholds from `config.ts`

**Output**: `PASS`, `PASS-WITH-WARNINGS` (requires user approval), or `FAIL` with mismatch list.

---

##### TASK-16E | (Assigned per mismatch) | `P2-BLOCKER-BOARD`
**Objective**: Fix Phase 2 mismatches per Retry Protocol.

---

### PHASE 3 — Hardening + Demo Reliability

---

#### Layer 6 — Polish → Milestone M6

##### TASK-17 | MiniMax M2.5 | `P3-HARDENING`
**Objective**: Wire error handling across the AI pipeline.

**Deliverables**:
- AI-1 retry: on failure, retry once with random jitter backoff (CONFIG.AI_RETRY_BACKOFF_MIN_MS to MAX_MS)
- Malformed AI output: one repair attempt (re-prompt with "return valid JSON only")
- Manual fallback: if AI-1 fails after retry, return empty extraction with `extractionStatus=FAILED` and raw text — confirmation screen handles manual entry
- Async pipeline failure: on AI-2/clustering error, set `clusteringStatus=FAILED` on Update — Blocker Board shows item as uncategorized/unclustered
- extractionStatus correctly set on all paths (SUCCESS, FAILED, MANUAL)

**Consumes**: `src/lib/ai/extract.ts`, `config.ts`

**PRD References**: Section 12 (Error Handling)

**Acceptance Criteria**: AI-1 failure → retry → fallback works. Malformed output triggers repair. clusteringStatus=FAILED items visible on Blocker Board as uncategorized.

---

##### TASK-18 | Kimi K2.5 | `P3-HARDENING`
**Objective**: Build demo seed button.

**Deliverables**:
- `src/app/api/demo/seed/route.ts` — `POST` (inserts 4–6 pre-extracted updates with realistic Done/Next/Blockers, creates BlockerItem records with hardcoded categories and **precomputed fixed embedding vectors** (hardcoded arrays — zero external API calls), creates 2–3 pre-built BlockerCluster records with precomputed centroid vectors, realistic aging and blast radius data)
- **Zero external API calls.** No LLM calls, no embedding API calls. All data is hardcoded for demo reliability. Embedding vectors are fixed arrays that are close enough in cosine similarity to trigger clustering logic correctly.
- UI button (in app header or standup page): "Seed Demo Data" — calls `POST /api/demo/seed`, refreshes views
- Confirmation dialog before seeding ("This will add sample data to the current project")

**Consumes**: `contracts.ts`, Prisma client

**PRD References**: Section 14.1 (Hackathon Demo Validation — "Seed demo updates" button)

**Acceptance Criteria**: One click populates Today Standup with 4–6 updates and Blocker Board with clustered blockers. No AI calls made. At least 1 recurring cluster with aging > 1 day and blast radius > 1 member.

---

##### TASK-19 | Kimi K2.5 | `P3-HARDENING`
**Objective**: Add Blocker Highlights footer to export.

**Deliverables**:
- Update `src/components/export-modal.tsx`
- Append compact footer to Markdown output: top N clusters (CONFIG.BLOCKER_HIGHLIGHTS_COUNT) sorted by aging, showing category, age in days, and blast radius
- Format matches Slack/Notion wrapper conventions

**Consumes**: `contracts.ts` (BlockerClustersResponse), `config.ts`

**PRD References**: Section 6.6 (Blocker Highlights footer)

**Acceptance Criteria**: Export includes footer with top 3 blocker clusters. Formatting correct for both Slack and Notion.

---

### DEFERRED — Do NOT build unless re-scoped

| Task | Agent | Priority | Description |
|------|-------|----------|-------------|
| TASK-20 | Kimi K2.5 | `DEFERRED` | Resurfaced badge tooltip with full resolution/resurfacing history timeline. |
| TASK-21 | Kimi K2.5 | `DEFERRED` | deviceId conflict warning UI: non-blocking warning when same memberName from different deviceIds. |
| TASK-22 | GLM-5 | `DEFERRED` | Quality score computation: rolling average over last N submissions per member. |
| TASK-23 | Kimi K2.5 | `DEFERRED` | By-Person export view toggle inside export modal. |

---

## Task Summary

| Phase | Tasks | Agent Distribution |
|-------|-------|--------------------|
| **Phase 1** (P1-CRITICAL) | 12 tasks (TASK-00 through TASK-16C) | MiniMax: 4, GLM-5: 5, Kimi: 5 |
| **Phase 2** (P2-BLOCKER-BOARD) | 7 tasks (TASK-06, 08, 09, 03B, 13, 16D, 16E) | MiniMax: 1, GLM-5: 4, Kimi: 1 |
| **Phase 3** (P3-HARDENING) | 3 tasks (TASK-17, 18, 19) | MiniMax: 1, Kimi: 2 |
| **Deferred** | 4 tasks (TASK-20 through 23) | GLM-5: 1, Kimi: 3 |
| **Total** | **26 tasks** | MiniMax: 6, GLM-5: 10, Kimi: 11 |

---

## Pre-Planned Endpoints (complete registry)

No endpoints may be added beyond this list after Phase 1 deploys.

| Endpoint | Method | Phase | Description |
|----------|--------|-------|-------------|
| `/api/projects` | POST | 1 | Create project |
| `/api/projects` | GET | 1 | List projects |
| `/api/updates` | POST | 1 | Submit raw text → AI extraction + quality check |
| `/api/updates/confirm` | POST | 1 | Store confirmed/edited update |
| `/api/standup` | GET | 1 | Today standup aggregation |
| `/api/blocker-clusters` | GET | 2 | Clusters with aging + blast radius |
| `/api/blocker-clusters/[id]/status` | PATCH | 2 | Toggle Open/Resolved |
| `/api/pipeline/run` | POST | 2 | Trigger async categorization + clustering (body: `{ updateId }`) |
| `/api/demo/seed` | POST | 3 | Insert pre-extracted demo data |
