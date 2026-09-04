# RootMind — Database Schema (In-Depth Edition)

**Document Version:** 3.0 (Deep Expansion)
**Database:** PostgreSQL (version 15+ assumed)
**ORM:** Prisma
**Companion to:** 01_PRD.md, 02_TECH_SPEC.md, 03_APP_FLOW.md, 06_IMPLEMENTATION_PLAN.md

---

## 1. Purpose and Scope of This Document

This document is the complete, authoritative reference for RootMind's data layer. It goes beyond simply listing tables and fields — it explains the reasoning behind every modeling decision, walks through how each table is actually queried in practice, estimates realistic data volumes over the project's lifetime, documents validation rules that live at the application layer (since Postgres/Prisma alone won't enforce all of them), and includes concrete example data so that anyone reading this document can picture exactly what the data looks like in production, not just what the schema definition says in the abstract.

Per the project's Rules document, this file must be kept in sync with the real schema at all times — any migration applied to the actual database must be reflected here in the same work session it was made.

---

## 2. Entity Relationship Overview

### 2.1 Diagram

```
                         ┌────────────┐
                         │    User     │
                         └─────┬──────┘
                               │ 1
                               │ many
                         ┌─────▼──────┐
                         │  Endpoint   │
                         └─────┬──────┘
                    ┌──────────┼──────────┐
                    │ 1                    │ 1
                    │ many                 │ many
              ┌─────▼──────┐        ┌──────▼───────┐
              │   Check     │        │   Incident    │
              └────────────┘        └──────┬───────┘
                                            │ 1
                                            │ many
                                     ┌──────▼───────┐
                                     │  Diagnosis    │
                                     └──────────────┘


        (Fully independent subsystem — no FK to anything above)

                    ┌──────────────────┐
                    │  EvaluationRun     │
                    └─────────┬────────┘
                              │ 1
                              │ many
                    ┌─────────▼────────┐
                    │ EvaluationResult   │
                    └───────────────────┘
```

### 2.2 Cardinality Table

| Parent | Child | Cardinality | Real-World Meaning |
|---|---|---|---|
| User | Endpoint | 1 : many | A user registers as many endpoints as they want to monitor |
| Endpoint | Check | 1 : many | Every scheduled health check produces exactly one new row, forever, for as long as the endpoint is active |
| Endpoint | Incident | 1 : many | An endpoint can go down and recover many separate times over its lifetime |
| Incident | Diagnosis | 1 : many | Normally one diagnosis per incident (the production default engine), but more than one when a user manually compares engines on the same real incident |
| EvaluationRun | EvaluationResult | 1 : many | Each run produces one result row per (labeled example × engine) combination |

### 2.3 Why No Relationship Exists Between the Evaluation Subsystem and Everything Else

This is worth stating explicitly because it's easy to assume, incorrectly, that `EvaluationResult` rows should reference real `Incident` rows. They deliberately do not. The evaluation subsystem operates entirely on a separate, versioned, file-based labeled dataset (see Section 8) so that:

- Research results remain reproducible independent of whatever real incidents happen to exist in the live database at any given moment.
- A user deleting or modifying a real endpoint/incident can never silently invalidate or corrupt historical evaluation results.
- The evaluation harness can be run repeatedly, safely, against a fixed dataset, without any risk of double-counting or interfering with production incident statistics shown to end users.

---

## 3. Full Prisma Schema (Annotated)

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// =================================================================
// User
// =================================================================
model User {
  id                String     @id @default(uuid())
  email             String     @unique
  passwordHash      String
  alertEmailEnabled Boolean    @default(true)
  createdAt         DateTime   @default(now())
  updatedAt         DateTime   @updatedAt

  endpoints         Endpoint[]

  @@map("users")
}

// =================================================================
// Endpoint
// =================================================================
model Endpoint {
  id               String     @id @default(uuid())
  userId           String
  user             User       @relation(fields: [userId], references: [id])

  name             String
  url              String
  httpMethod       String     @default("GET")
  expectedStatus   Int        @default(200)
  checkIntervalSec Int        @default(60)
  timeoutMs        Int        @default(5000)
  customHeaders    Json?

  isActive         Boolean    @default(true)
  nextCheckAt      DateTime   @default(now())

  createdAt        DateTime   @default(now())
  updatedAt        DateTime   @updatedAt

  checks           Check[]
  incidents        Incident[]

  @@index([userId])
  @@index([nextCheckAt])
  @@map("endpoints")
}

// =================================================================
// Check
// =================================================================
model Check {
  id               String     @id @default(uuid())
  endpointId       String
  endpoint         Endpoint   @relation(fields: [endpointId], references: [id])

  checkedAt        DateTime   @default(now())
  success          Boolean
  statusCode       Int?
  responseTimeMs   Int?
  errorType        String?

  responseSnippet  String?
  responseHeaders  Json?

  @@index([endpointId, checkedAt])
  @@map("checks")
}

// =================================================================
// Incident
// =================================================================
model Incident {
  id           String      @id @default(uuid())
  endpointId   String
  endpoint     Endpoint    @relation(fields: [endpointId], references: [id])

  status       String      @default("open")
  openedAt     DateTime    @default(now())
  resolvedAt   DateTime?
  durationSec  Int?

  diagnoses    Diagnosis[]

  @@index([endpointId, status])
  @@map("incidents")
}

// =================================================================
// Diagnosis
// =================================================================
model Diagnosis {
  id                String    @id @default(uuid())
  incidentId        String
  incident          Incident  @relation(fields: [incidentId], references: [id])

  engineUsed        String
  status            String    @default("completed")

  rankedCauses      Json
  explanation       String?
  suggestedNextStep String?

  latencyMs         Int?
  createdAt         DateTime  @default(now())

  @@index([incidentId])
  @@map("diagnoses")
}

// =================================================================
// EvaluationRun
// =================================================================
model EvaluationRun {
  id             String              @id @default(uuid())
  startedAt      DateTime            @default(now())
  completedAt    DateTime?
  datasetSize    Int
  datasetVersion String?
  notes          String?

  results        EvaluationResult[]

  @@map("evaluation_runs")
}

// =================================================================
// EvaluationResult
// =================================================================
model EvaluationResult {
  id                String        @id @default(uuid())
  evaluationRunId   String
  evaluationRun     EvaluationRun @relation(fields: [evaluationRunId], references: [id])

  engineUsed        String
  labeledExampleId  String
  groundTruthCause  String
  predictedCauses   Json
  top1Correct       Boolean
  top3Correct       Boolean
  latencyMs         Int

  createdAt         DateTime @default(now())

  @@index([evaluationRunId, engineUsed])
  @@map("evaluation_results")
}
```

**Note on `@@map`:** each model is explicitly mapped to a `snake_case`, pluralized table name at the database level (`users`, `endpoints`, `checks`, etc.), while Prisma model names stay `PascalCase` and singular in application code. This is a common, deliberate convention: it keeps the generated SQL and any manual `psql` inspection during debugging readable in the style most database tooling expects, while keeping the JavaScript/TypeScript side of the codebase idiomatic to that ecosystem.

---

## 4. Field-by-Field Reference Tables

### 4.1 `User`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `email` | String | No | — | Unique constraint enforced at the DB level; format validated at the application layer before insert |
| `passwordHash` | String | No | — | Never the raw password; bcrypt hash only, generated with a cost factor of at least 10 |
| `alertEmailEnabled` | Boolean | No | `true` | Controls whether incident open/resolve emails are sent |
| `createdAt` | DateTime | No | now() | Set once, never updated |
| `updatedAt` | DateTime | No | auto | Updated automatically by Prisma on every write to this row |

**Application-layer validation not enforced by the schema itself:**
- Email must match a standard email-format regex before insert.
- Password must meet a minimum length (e.g., 8 characters) before being hashed — this check happens before `passwordHash` is ever computed, since the raw password is never persisted in any form.

### 4.2 `Endpoint`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `userId` | UUID | No | — | Foreign key to `User.id` |
| `name` | String | No | — | Free text, shown throughout the UI |
| `url` | String | No | — | Full URL including protocol; validated as a well-formed URL at the application layer |
| `httpMethod` | String | No | `"GET"` | Expected values: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD` — validated at the application layer against this allow-list |
| `expectedStatus` | Int | No | `200` | Any valid HTTP status code |
| `checkIntervalSec` | Int | No | `60` | Constrained at the application layer to one of the UI's offered values (60, 300, 900) to avoid a user accidentally configuring an unreasonably aggressive interval |
| `timeoutMs` | Int | No | `5000` | Reasonable bounds (e.g., 1000–30000ms) enforced at the application layer |
| `customHeaders` | JSON | Yes | `null` | Object of string key-value pairs; validated as valid JSON and reasonable size at the application layer |
| `isActive` | Boolean | No | `true` | `false` represents both "paused by the user" and "soft-deleted" — see Section 4.2.1 below for why these share one flag |
| `nextCheckAt` | DateTime | No | now() | Updated by the scheduler after every check attempt; this is the field the scheduler's core query filters on |

#### 4.2.1 Why "Paused" and "Soft-Deleted" Share the Same `isActive` Flag

Functionally, a paused endpoint and a deleted endpoint behave identically from the scheduler's point of view — neither should be checked. Introducing a separate `deletedAt` field alongside `isActive` was considered but rejected for this project's scope: it would add a second piece of state to keep in sync everywhere `isActive` is checked, for a distinction (paused vs. deleted) that only matters for how the endpoint is labeled in the UI, not for any behavioral difference in the monitoring engine itself. If a future need arose to distinguish them (e.g., showing "deleted" endpoints differently from "paused" ones in some admin view), a `deletedAt` field could be added at that point without disrupting the existing `isActive` behavior.

### 4.3 `Check`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `endpointId` | UUID | No | — | Foreign key to `Endpoint.id` |
| `checkedAt` | DateTime | No | now() | The moment this check was executed |
| `success` | Boolean | No | — | The single most important field for uptime percentage calculations |
| `statusCode` | Int | Yes | `null` | Null when the request failed before receiving any response (e.g., timeout, DNS failure) |
| `responseTimeMs` | Int | Yes | `null` | Null only in the rare case a duration genuinely couldn't be measured (e.g., DNS resolution failure before any connection attempt began) |
| `errorType` | String | Yes | `null` | Populated only when `success = false`; one of the defined taxonomy values (see Section 6) |
| `responseSnippet` | String | Yes | `null` | Populated only on failure; truncated to a reasonable length (e.g., first 500 characters) at write time to bound row size |
| `responseHeaders` | JSON | Yes | `null` | Populated only on failure |

### 4.4 `Incident`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `endpointId` | UUID | No | — | Foreign key to `Endpoint.id` |
| `status` | String | No | `"open"` | Exactly two valid values: `"open"`, `"resolved"` |
| `openedAt` | DateTime | No | now() | Set once, at creation |
| `resolvedAt` | DateTime | Yes | `null` | Set exactly once, when the incident resolves; remains `null` while open |
| `durationSec` | Int | Yes | `null` | Calculated as `resolvedAt - openedAt` in seconds, computed and stored at resolution time rather than calculated on every read, to keep incident-list queries fast |

### 4.5 `Diagnosis`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `incidentId` | UUID | No | — | Foreign key to `Incident.id` |
| `engineUsed` | String | No | — | Exactly one of: `"prompt"`, `"ml"`, `"finetuned"` |
| `status` | String | No | `"completed"` | One of: `"completed"`, `"unavailable"`, `"pending"` |
| `rankedCauses` | JSON | No | — | Array of `{ cause: string, confidence: number }`, sorted descending by confidence; empty array `[]` when `status = "unavailable"` |
| `explanation` | String | Yes | `null` | Null only when `status = "unavailable"` |
| `suggestedNextStep` | String | Yes | `null` | Null only when `status = "unavailable"` |
| `latencyMs` | Int | Yes | `null` | Time taken by the engine; null if the call never returned at all (pure timeout with no partial response) |
| `createdAt` | DateTime | No | now() | When this diagnosis attempt was recorded |

### 4.6 `EvaluationRun`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `startedAt` | DateTime | No | now() | — |
| `completedAt` | DateTime | Yes | `null` | Null while the run is still in progress |
| `datasetSize` | Int | No | — | Total number of labeled examples used in this run |
| `datasetVersion` | String | Yes | `null` | E.g. `"v1"`, `"v2"` — ties results back to a specific dataset file |
| `notes` | String | Yes | `null` | Free text, e.g. "First full 3-engine comparison run, held-out test split only" |

### 4.7 `EvaluationResult`

| Field | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | UUID | No | random UUID | Primary key |
| `evaluationRunId` | UUID | No | — | Foreign key to `EvaluationRun.id` |
| `engineUsed` | String | No | — | Exactly one of: `"prompt"`, `"ml"`, `"finetuned"` |
| `labeledExampleId` | String | No | — | References an ID within the external labeled dataset file, not a foreign key into any live table |
| `groundTruthCause` | String | No | — | The known-correct cause for this labeled example |
| `predictedCauses` | JSON | No | — | Same shape as `Diagnosis.rankedCauses` |
| `top1Correct` | Boolean | No | — | Whether the engine's single top-ranked cause matched ground truth exactly |
| `top3Correct` | Boolean | No | — | Whether ground truth appeared anywhere in the engine's top three ranked causes |
| `latencyMs` | Int | No | — | Time taken for this specific diagnosis call |

---

## 5. Realistic Data Volume Estimates

Understanding roughly how large each table will grow helps justify the indexing strategy (Section 7) and the retention discussion (Section 9). Assuming a moderate, realistic usage pattern for a student project over one semester (roughly 14 weeks of active monitoring):

| Table | Assumptions | Estimated Row Count by End of Semester |
|---|---|---|
| `User` | A handful of accounts (the author, possibly teammates for testing) | Under 10 |
| `Endpoint` | 10–20 endpoints monitored across real and test/chaos-engineering services | 10–30 |
| `Check` | 15 endpoints average, checked every 60 seconds, 24/7, for 14 weeks | 15 endpoints × 1,440 checks/day × 98 days ≈ **2.1 million rows** |
| `Incident` | Assuming a generous 2% of checks result in a new incident being opened (most failures are part of an ongoing incident, not a new one) | Roughly 500–2,000 rows, depending on how much deliberate chaos-testing is done |
| `Diagnosis` | Roughly one per incident in normal operation, plus extra rows from manual multi-engine comparisons during development/demos | Similar order of magnitude to `Incident`, perhaps 1.5–2x due to manual re-runs |
| `EvaluationRun` | A handful of full evaluation harness runs over the project's life (initial run, re-runs after dataset improvements, final run for the report) | Under 20 |
| `EvaluationResult` | Each run processes the full labeled dataset (estimated 200–500 examples) × 3 engines | Roughly 600–1,500 rows per run |

**Implication:** `Check` is overwhelmingly the largest table by a wide margin — multiple orders of magnitude larger than every other table combined. This directly justifies both the indexing priority given to `Check(endpointId, checkedAt)` and the data retention discussion in Section 9; it is also worth explicitly mentioning this estimate during a viva if asked about the system's scalability characteristics, since it demonstrates the author has thought about real-world data growth, not just the schema's structure in isolation.

---

## 6. The Failure Taxonomy (Canonical List)

This taxonomy is referenced by `Check.errorType`, by the Engine 1 prompt's category list, by Engine 2's classification labels, and by Engine 3's fine-tuning data — it must stay perfectly consistent across all four, per the Rules document's Section 5. This schema document is the canonical place this list is defined; all other documents and code should reference it from here rather than redefining it independently.

| Taxonomy Value | Meaning | Typical Signal Pattern |
|---|---|---|
| `TIMEOUT` | The request did not complete within the configured timeout | Often preceded by a gradually climbing `responseTimeMs` trend in `recentHistory` |
| `CONNECTION_REFUSED` | The target server actively refused the connection | Usually a sudden, hard failure with no preceding latency creep |
| `DNS_FAILURE` | The endpoint's hostname could not be resolved | Often affects multiple endpoints sharing the same domain simultaneously |
| `SSL_ERROR` | A TLS/SSL handshake or certificate validation failure | Frequently correlates with a known certificate expiry date, if historical data is available |
| `NON_2XX` | The request completed, but returned a status code other than the endpoint's `expectedStatus` | The specific status code (e.g., 500, 429, 404) is itself a strong diagnostic signal, further sub-classified within the diagnosis engines' reasoning rather than as a separate taxonomy value here |
| `DB_POOL_EXHAUSTION` *(inferred, not a raw error type)* | Not directly observable from a single check's `errorType`, but inferred by the diagnosis engines from a `TIMEOUT`/`NON_2XX` pattern combined with a specific latency-creep shape | Included here because it's a common ground-truth label in the labeled dataset (Section 8), even though it is a diagnosis-engine *output* category rather than a raw `Check.errorType` value |
| `RATE_LIMITED` *(inferred from status code)* | Derived when `statusCode = 429` | — |
| `PARTIAL_OUTAGE_CORRELATED` *(inferred, diagnosis-only)* | Not a `Check.errorType` — a diagnosis-level conclusion drawn when `correlatedFailures` shows multiple unrelated endpoints failing in the same window | — |

**Important distinction:** the first five rows are genuine `Check.errorType` values, directly observable from a single failed request. The last three are *diagnosis-level conclusions* — they only ever appear as values inside `Diagnosis.rankedCauses` or as `EvaluationResult.groundTruthCause`/`predictedCauses` values, never as a raw `Check.errorType`. Keeping this distinction clear in the schema (rather than trying to cram diagnostic conclusions into the same field as raw error classification) is what keeps `Check` an honest record of what was directly observed, separate from what was later inferred about it.

---

## 7. Indexing Strategy — Query-by-Query Justification

### 7.1 `Endpoint(userId)`

**Serves:** `SELECT * FROM endpoints WHERE user_id = ? AND is_active = true` — the Dashboard's primary load query, executed on every Dashboard page view and every 30-second polling refresh.

**Why it matters:** without this index, as the total number of endpoints across all users grows, this becomes a full table scan on every single dashboard load for every user — directly violating the PRD's sub-2-second dashboard load success metric.

### 7.2 `Endpoint(nextCheckAt)`

**Serves:** `SELECT * FROM endpoints WHERE is_active = true AND next_check_at <= NOW()` — the scheduler's core query, executed every 30 seconds regardless of how many users or endpoints exist.

**Why it matters:** this is the single hottest query path in the entire system by execution frequency. A slow scheduler query directly translates into check delays, which directly undermines the accuracy of uptime tracking — the system's core purpose.

### 7.3 `Check(endpointId, checkedAt)` — Compound Index

**Serves two distinct query shapes:**
1. `SELECT * FROM checks WHERE endpoint_id = ? ORDER BY checked_at DESC LIMIT 20` — the recent-checks table on Endpoint Detail.
2. `SELECT * FROM checks WHERE endpoint_id = ? AND checked_at >= ? ORDER BY checked_at ASC` — the response-time graph's data range query.

**Why a compound index, not two single-column indexes:** both query shapes filter by `endpointId` first, and then need results ordered by `checkedAt`. A compound index with `endpointId` as the leading column lets Postgres jump straight to that endpoint's rows and then read them in the already-sorted order the index maintains, avoiding a separate sort step entirely.

### 7.4 `Incident(endpointId, status)`

**Serves:** `SELECT * FROM incidents WHERE endpoint_id = ? AND status = 'open'` — checked by the Incident Manager on literally every failed or succeeded check, to decide whether an open incident already exists or needs to be created/resolved.

**Why it matters:** this check happens at the same frequency as check processing itself (every 30-second tick, for every endpoint checked in that tick), so it needs to stay fast even as the total historical incident count grows into the thousands over the semester.

### 7.5 `Diagnosis(incidentId)`

**Serves:** `SELECT * FROM diagnoses WHERE incident_id = ? ORDER BY created_at DESC` — the Incident Detail page's Diagnosis Panel query, which needs to retrieve all diagnosis attempts (potentially from multiple engines) for a single incident.

### 7.6 `EvaluationResult(evaluationRunId, engineUsed)`

**Serves:** `SELECT * FROM evaluation_results WHERE evaluation_run_id = ? AND engine_used = ?` (and aggregate variants: `GROUP BY engine_used` within a run) — every metric shown on the Evaluation Results page is computed per-engine, within a specific run, so this compound index matches that access pattern exactly.

---

## 8. The External Labeled Dataset File (Referenced by, but Not Stored in, the Database)

Section 4.7 and Section 2.3 both reference an external labeled dataset that the evaluation subsystem depends on. This dataset lives as a versioned JSON file (e.g., `/data/labeled_incidents_v1.json`), not as database rows, specifically so it can be version-controlled in Git alongside the code, diffed between versions, and included directly as a reproducibility artifact in the final report or paper. Its structure mirrors the Diagnosis Context Schema from the Tech Spec, with a ground-truth label attached:

```json
[
  {
    "labeledExampleId": "synthetic-0001",
    "contextPayload": {
      "incidentId": "N/A - synthetic",
      "endpoint": { "url": "https://test-service.local/api/orders", "method": "GET", "expectedStatus": 200 },
      "failureSignal": { "statusCode": null, "errorType": "TIMEOUT", "responseTimeMs": 5000, "responseSnippet": null },
      "recentHistory": [
        { "timestamp": "2026-08-10T14:00:00Z", "success": true, "responseTimeMs": 220, "statusCode": 200 },
        { "timestamp": "2026-08-10T14:01:00Z", "success": true, "responseTimeMs": 850, "statusCode": 200 },
        { "timestamp": "2026-08-10T14:02:00Z", "success": false, "responseTimeMs": 5000, "statusCode": null }
      ],
      "correlatedFailures": [],
      "historicalFailureRate": 0.02
    },
    "groundTruthCause": "DB_POOL_EXHAUSTION",
    "generationMethod": "synthetic"
  }
]
```

- `generationMethod` records whether the example came from a real chaos-engineering session (`"chaos"`) or was scripted (`"synthetic"`) — this distinction is worth preserving and potentially reporting separately in the evaluation results, since real chaos-derived examples arguably carry more validity than synthetic ones, and a rigorous report should be transparent about that split rather than presenting all examples as equally strong evidence.

---

## 9. Data Retention Considerations (Expanded)

Given the volume estimate in Section 5 (roughly 2 million `Check` rows by the end of the semester), an unmanaged `Check` table is the one genuine long-term scalability concern in this schema. Two possible approaches, documented here for completeness even if only partially implemented given time constraints:

**Option A — Time-based aggregation (preferred, if implemented):** `Check` rows older than a configurable window (e.g., 30 or 90 days) are periodically aggregated into a new, much smaller `CheckDailySummary` table (one row per endpoint per day: total checks, total failures, min/avg/max response time), after which the original fine-grained rows are deleted. This preserves long-term uptime-percentage history for the dashboard's historical views while keeping the actively-queried portion of `Check` bounded in size.

**Option B — Simple deletion (fallback, minimal effort):** `Check` rows older than a fixed window are deleted outright with no aggregation, sacrificing long-term historical detail in exchange for zero additional implementation work.

**Recommendation for this project's timeline:** given that the Implementation Plan already carries meaningful scope across seven phases, Option B (or no pruning at all, relying on the fact that a single semester's data volume, while large, is still well within what a single PostgreSQL instance can comfortably handle) is the pragmatic choice — but Option A is worth describing in the final report as a "if this were a production system" forward-looking design consideration, since it demonstrates awareness of the tradeoff without requiring the engineering time to build it.

---

## 10. Backup and Recovery (Minimal, Project-Appropriate Scope)

Given this is a student project rather than a production system with real users depending on uptime guarantees, a full disaster-recovery strategy is out of scope. The pragmatic minimum worth doing:

- Rely on the hosting provider's built-in automated backups where available (Render/Railway both offer basic managed Postgres backups on their free/low tiers).
- Before any risky schema migration (especially anything destructive, like a column type change), take a manual export (`pg_dump`) as a safety net.
- Keep the labeled dataset file (Section 8) version-controlled in Git — this is arguably more important to protect than the live database, since it represents irreplaceable manual labeling/chaos-engineering effort, whereas the live `Check`/`Incident` history, while valuable, would simply begin re-accumulating from the next scheduler tick if lost.

---

## 11. Schema Evolution Log

*(Updated whenever a migration is applied, per the Rules document's Documentation Upkeep rule.)*

| Date | Change | Reason |
|---|---|---|
| Initial | `User`, `Endpoint`, `Check` models created | Phase 1 — core monitoring engine |
| — | `Incident` model added | Phase 1 — incident lifecycle tracking |
| — | `Diagnosis` model added | Phase 2 — Engine 1 (prompt-based) integration |
| — | `EvaluationRun`, `EvaluationResult` models added | Phase 6 — comparative evaluation harness |
| — | `datasetVersion` field added to `EvaluationRun` | Added to satisfy the research-integrity requirement that every reported metric be traceable to a specific dataset version |
