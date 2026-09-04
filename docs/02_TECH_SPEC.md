# RootMind — Technical Specification

**Document Version:** 2.0 (Expanded)
**Companion to:** 01_PRD.md
**Audience:** Developer(s)/agent implementing RootMind, and the mentor/panel reviewing technical decisions

---

## 1. Purpose of This Document

Where the PRD defines *what* RootMind must do and *why*, this Tech Spec defines *how* it will be built: the technology choices and the reasoning behind each one, the system's component architecture, the API surface, the shared data contract between the three AI diagnosis engines, non-functional requirements, coding conventions, and the full set of third-party dependencies. Anyone implementing a feature should be able to answer "where does this code belong, and how does it talk to the rest of the system" by reading this document alone.

---

## 2. Technology Stack — Full Reasoning

| Layer | Choice | Why This, Specifically |
|---|---|---|
| Backend runtime | Node.js (LTS version) | Prior hands-on familiarity from other projects (CollabSphere, EduSangam) means less ramp-up time; strong async I/O model fits a system whose core job is issuing many concurrent outbound HTTP requests (health checks) without blocking |
| Backend framework | Express.js | Minimal, unopinionated, well-documented — appropriate for a project where the value is in the monitoring/diagnosis logic, not in framework abstraction; keeps routing and middleware easy to reason about for a mentor reviewing the code |
| Database | PostgreSQL | Needs genuine relational integrity (a Check belongs to exactly one Endpoint, an Incident aggregates many Checks, a Diagnosis belongs to exactly one Incident) — a relational database models this naturally; also has strong support for time-ordered queries needed for uptime percentage and latency-trend calculations |
| ORM / Query layer | Prisma | Type-safe schema definition and auto-generated client reduce a whole class of query-shape bugs; built-in migration tooling keeps schema evolution auditable, which matters when the schema changes across project phases (e.g., adding Diagnosis and EvaluationRun tables later) |
| Job scheduling | node-cron | The health-check loop is a simple recurring tick, not a complex distributed job queue — a heavyweight solution (e.g., BullMQ + Redis) would add operational complexity with no real benefit at this project's scale; a DB-backed "next check due" query is sufficient and simpler to explain in a viva |
| LLM Provider (Engine 1, and inference for Engine 3) | Groq API (LLaMA 3 family) | Fast inference speed matters because diagnosis should feel near-real-time on the Incident Detail page; free/low-cost tier fits the zero-budget constraint; prior familiarity from earlier project work reduces integration risk |
| ML framework (Engine 2) | Python + scikit-learn / XGBoost | The most standard, well-documented tooling for tabular classification problems; scikit-learn's built-in cross-validation and metrics utilities directly support the rigorous evaluation the mentor has asked for; XGBoost is included specifically because gradient-boosted trees are frequently the strongest performer on structured/tabular data of the kind this project generates |
| Fine-tuning (Engine 3) | Hugging Face Transformers + PEFT (LoRA/QLoRA) | LoRA/QLoRA make fine-tuning feasible on free-tier GPU compute (Colab/Kaggle) by only training a small number of additional parameters rather than the full model — this is what makes Engine 3 achievable at all within a zero-budget, no-dedicated-GPU constraint |
| Frontend | React (with Vite as the build tool) | Component-based structure fits a dashboard made of clearly reusable pieces (status badges, incident cards, charts); Vite's fast dev server keeps the iteration loop quick during a busy semester |
| Styling | Plain CSS / CSS Modules | Deliberately avoids a heavy UI component library (e.g., Material UI, Ant Design) whose default look would fight against the minimalist design requirement in 04_DESIGN.md; hand-written CSS keeps full control over the calm, restrained visual language the project calls for |
| Charts | Recharts | Lightweight, React-native charting library, sufficient for the simple line/bar charts this project needs (response-time trends, evaluation comparisons) without pulling in a large, general-purpose visualization framework |
| Auth | Custom JWT-based email/password auth | Third-party auth providers (Auth0, Firebase Auth, Clerk) add external dependencies and setup overhead disproportionate to this project's single-user-account scope; a hand-rolled JWT flow is also a well-understood, demonstrable piece of engineering for a viva |
| Notifications | Nodemailer (SMTP) | Simple, free, well-documented; matches the PRD's deliberately minimal alerting scope (email only, no paid notification services) |
| Deployment | Render or Railway for the primary deployment; AWS EC2 as an optional stretch goal | Render/Railway offer free/low-cost tiers with minimal DevOps overhead, appropriate given time constraints; EC2 is kept as an explicit stretch goal specifically because deploying there is a valuable, separate skill-building opportunity, not because it's required for the project's core grading criteria |
| Version control | Git + GitHub | Standard; also enables straightforward collaboration with Neural Nomads teammates if any part of the codebase is shared |

### 2.1 Why Node.js/PostgreSQL for the Core, but Python for the ML/Fine-Tuning Pipelines

This is a deliberate hybrid architecture, not an inconsistency. The core product — monitoring engine, API, database, dashboard — is confirmed and built in Node.js/Express/PostgreSQL, matching the project's established stack and the author's backend-leaning strengths. However, Engines 2 and 3 depend on Python's considerably more mature ML/fine-tuning ecosystem (scikit-learn, XGBoost, Hugging Face Transformers, PEFT) — rewriting equivalent tooling in JavaScript would be both harder and less credible academically, since it would mean reinventing well-established libraries rather than using them correctly. The two halves of the system communicate over a small internal HTTP contract: the Node.js backend calls out to lightweight Python inference services (FastAPI/Flask) for Engines 2 and 3, using the exact same shared diagnosis contract as Engine 1's in-process Node.js implementation. This keeps the "core system" honestly described as Node.js/PostgreSQL while still using the right tool for each AI approach.

---

## 3. System Architecture

### 3.1 High-Level Component Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                              RootMind System                          │
│                                                                        │
│   ┌────────────────┐        ┌─────────────────────────┐              │
│   │   React SPA     │◄──────►│    Express API Server    │              │
│   │  (Dashboard UI) │  HTTP  │       (Node.js)          │              │
│   └────────────────┘        └────────────┬─────────────┘              │
│                                           │                            │
│                 ┌─────────────────────────┼─────────────────────────┐  │
│                 │                         │                         │  │
│         ┌───────▼────────┐     ┌──────────▼──────────┐    ┌─────────▼──────────┐
│         │  Health Check   │     │   Incident Manager   │    │   Diagnosis         │
│         │  Scheduler       │     │                       │    │   Orchestrator      │
│         └───────┬────────┘     └──────────┬──────────┘    └─────────┬──────────┘
│                 │                         │                         │            │
│                 └────────────┬────────────┴────────────┬────────────┘            │
│                              │                         │                         │
│                    ┌─────────▼─────────┐     ┌─────────▼─────────────────────┐    │
│                    │   PostgreSQL DB    │     │      Diagnosis Engines          │    │
│                    │   (via Prisma)     │     │  ┌──────────────────────────┐  │    │
│                    └───────────────────┘     │  │ Engine 1: Prompt-based    │  │    │
│                                               │  │ (in-process, Groq API)    │  │    │
│                                               │  └──────────────────────────┘  │    │
│                                               │  ┌──────────────────────────┐  │    │
│                                               │  │ Engine 2: Custom ML       │  │    │
│                                               │  │ (Python FastAPI service)  │  │    │
│                                               │  └──────────────────────────┘  │    │
│                                               │  ┌──────────────────────────┐  │    │
│                                               │  │ Engine 3: Fine-tuned LLM  │  │    │
│                                               │  │ (Python FastAPI service)  │  │    │
│                                               │  └──────────────────────────┘  │    │
│                                               └────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Request Lifecycle Example — A Failing Endpoint, End to End

To make the architecture concrete, here is exactly what happens, step by step, when a monitored endpoint fails:

1. The Health Check Scheduler's tick fires (every 30 seconds) and finds that a given endpoint's `nextCheckAt` timestamp has passed.
2. It issues an HTTP request to that endpoint's URL, using its configured method, headers, and timeout.
3. The request fails (say, it times out). The scheduler writes a `Check` row: `success = false`, `errorType = "TIMEOUT"`, `responseTimeMs = <timeout value>`, `statusCode = null`.
4. The scheduler updates the endpoint's `nextCheckAt` to the next scheduled time and hands the failed check off to the Incident Manager.
5. The Incident Manager checks: has this endpoint now failed 2 (default threshold) times in a row? If this is only the first failure, nothing further happens yet — it waits for the next check.
6. Suppose it is the second consecutive failure. The Incident Manager creates a new `Incident` row with `status = "open"`.
7. The Incident Manager immediately (but asynchronously — not blocking the scheduler's next tick) invokes the Diagnosis Orchestrator with the new incident's ID.
8. The Diagnosis Orchestrator builds the structured context payload: the failure signal itself, the last several checks for this endpoint (to see the latency trend leading up to the timeout), whether any other monitored endpoints failed in the same time window, and this endpoint's historical failure rate.
9. The Orchestrator calls the currently-active diagnosis engine (Engine 1 by default in production use) with this payload.
10. Engine 1 sends the payload to the Groq API inside a structured prompt, receives a JSON response, and the Orchestrator validates and parses it into the shared diagnosis format.
11. The Orchestrator saves a `Diagnosis` row linked to the incident, including the ranked causes, explanation, suggested next step, and how long the engine took to respond.
12. If email alerting is enabled, an email is sent to the user summarizing the incident and (if the diagnosis completed in time) the top probable cause.
13. On the frontend, the next time the Dashboard or Incident Detail page polls the API, it reflects the new "Down" status and, shortly after, the diagnosis result.
14. Later, when a scheduled check for this endpoint finally succeeds again, the Incident Manager marks the incident `resolved`, calculates its duration, and (if enabled) sends a resolution email.

---

## 4. Component Responsibilities (Detailed)

### 4.1 Health Check Scheduler

- **Trigger:** A `node-cron` job configured to tick every 30 seconds (chosen to be short enough that even 1-minute check intervals stay reasonably accurate, without polling so frequently that it wastes resources).
- **Core query:** `SELECT * FROM Endpoint WHERE isActive = true AND nextCheckAt <= NOW()`.
- **Per-endpoint execution:** issues the configured HTTP request using `axios`, respecting the endpoint's method, headers, and timeout; catches and classifies errors into the defined `errorType` categories rather than letting raw exceptions propagate.
- **Write-back:** always writes a `Check` row, regardless of outcome, and always updates `nextCheckAt`, even on failure — a failing endpoint must keep being checked on schedule, not silently stop.
- **Isolation:** one endpoint's check failing or throwing an unexpected error must never prevent other endpoints in the same tick from being checked — each endpoint's check is wrapped in its own try/catch.
- **Heartbeat logging:** logs a single line per tick summarizing how many endpoints were checked, so both the developer and, during a demo, the mentor can see the scheduler is alive and functioning.

### 4.2 Incident Manager

- **Consecutive-failure threshold:** configurable per-deployment constant (default: 2), applied per endpoint independently.
- **Flap-cooldown:** if an endpoint resolves and then fails again within a short window (e.g., 2 minutes), the new failure reopens/extends the prior incident rather than creating a brand-new one, avoiding a cluttered incident history from genuinely flaky (rather than truly down) endpoints.
- **State transitions:** the only two incident states are `open` and `resolved` — deliberately simple, since more granular states (acknowledged, investigating, monitoring, etc., as full incident-management tools use) are explicitly out of scope per the PRD's non-goals.

### 4.3 Diagnosis Orchestrator

- **Context building:** a single, shared function (`buildDiagnosisContext(incidentId)`) is the *only* place in the codebase allowed to construct the context payload — this guarantees all three engines truly receive identical information, which is essential for a fair comparison.
- **Engine selection:** in normal production operation, only the configured default engine (Engine 1) runs automatically per incident. All three engines can be run on-demand from the Incident Detail page (for live demo comparison) or via the Evaluation Harness (for the dataset-wide comparison).
- **Timeout and fallback handling:** each engine call has its own timeout; if it fails or times out (including after retries, for Engine 1's malformed-JSON retry logic), the Orchestrator persists a `Diagnosis` row with `status = "unavailable"` rather than leaving the incident without any diagnosis record at all — the absence of a successful diagnosis is itself recorded, not silently dropped.

### 4.4 Diagnosis Engines — Shared Contract

All three engines, regardless of implementation language or approach, must satisfy exactly this interface:

```
diagnose(contextPayload) -> {
  engineUsed: "prompt" | "ml" | "finetuned",
  rankedCauses: [ { cause: string, confidence: number } ],   // sorted, most confident first
  explanation: string,        // plain-English paragraph
  suggestedNextStep: string,  // one actionable suggestion
  latencyMs: number           // time this specific engine took to respond
}
```

This shared contract is the single most important architectural decision in the whole system: it is what makes the three-way comparison in the PRD's research objective actually valid. If any engine's output shape diverged, comparing them would not be a fair, like-for-like evaluation.

- **Engine 1 (Prompt-based):** implemented directly inside the Node.js backend (`/src/services/llmDiagnosisEngine.js`), since it requires no separate model-serving infrastructure — just an outbound API call.
- **Engine 2 (Custom ML):** implemented as a small Python FastAPI service exposing a single `/diagnose` POST route that accepts the context payload and returns the shared contract shape; the Node.js Orchestrator calls this service exactly the same way it would call any other HTTP dependency.
- **Engine 3 (Fine-tuned LLM):** implemented the same way as Engine 2 — a Python FastAPI service wrapping the fine-tuned model's inference call, again returning the shared contract shape.

### 4.5 Evaluation Harness

- Deliberately built as a **separate module**, not part of the live request path — it is triggered manually (via an admin-facing API route or a standalone script), not by user traffic.
- Reads from the versioned labeled dataset file (not the live `Check`/`Incident` tables), ensuring evaluation results are reproducible independent of whatever real incidents happen to exist in the database at any given time.
- Writes results into dedicated `EvaluationRun`/`EvaluationResult` tables (see 05_SCHEMA.md), which are never touched by any other part of the system.

---

## 5. API Endpoints (Complete)

| Method | Path | Auth Required | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | No | Create a new user account (email + password) |
| POST | `/api/auth/login` | No | Authenticate and receive a JWT |
| GET | `/api/endpoints` | Yes | List all endpoints belonging to the logged-in user |
| POST | `/api/endpoints` | Yes | Register a new endpoint (runs a one-time validation ping) |
| GET | `/api/endpoints/:id` | Yes | Retrieve one endpoint's configuration and summary stats |
| PUT | `/api/endpoints/:id` | Yes | Update an endpoint's configuration |
| DELETE | `/api/endpoints/:id` | Yes | Soft-delete (deactivate) an endpoint |
| GET | `/api/endpoints/:id/checks` | Yes | Paginated raw check history for one endpoint |
| GET | `/api/incidents` | Yes | List incidents across all of the user's endpoints, filterable by status |
| GET | `/api/incidents/:id` | Yes | Full incident detail, including any associated diagnoses |
| POST | `/api/incidents/:id/diagnose` | Yes | Manually trigger (or re-trigger) diagnosis for an incident; accepts an `engine` query parameter (`prompt`, `ml`, or `finetuned`) |
| GET | `/api/evaluation/results` | Yes (admin-level) | Retrieve stored results from the most recent (or a specified) evaluation run |
| POST | `/api/evaluation/run` | Yes (admin-level) | Trigger a full evaluation harness run against the labeled dataset |
| PUT | `/api/settings` | Yes | Update account-level settings (e.g., alert email toggle) |

Every route other than `/api/auth/*` requires a valid JWT, passed in the `Authorization: Bearer <token>` header, verified by a shared auth middleware before the request reaches its controller.

---

## 6. Diagnosis Context Schema — Full Definition

```json
{
  "incidentId": "string",
  "endpoint": {
    "url": "string",
    "method": "string",
    "expectedStatus": "number"
  },
  "failureSignal": {
    "statusCode": "number | null",
    "errorType": "string",
    "responseTimeMs": "number",
    "responseSnippet": "string | null"
  },
  "recentHistory": [
    {
      "timestamp": "ISO8601 string",
      "success": "boolean",
      "responseTimeMs": "number",
      "statusCode": "number"
    }
  ],
  "correlatedFailures": [
    {
      "endpointUrl": "string",
      "failedAtSameWindow": "boolean"
    }
  ],
  "historicalFailureRate": "number"
}
```

Every field here exists because it maps directly to something a human debugging an outage would want to check first: the failure signal answers "what exactly happened," `recentHistory` answers "was this sudden or a gradual decline," `correlatedFailures` answers "is this isolated or part of something bigger," and `historicalFailureRate` answers "is this a known flaky endpoint or a genuine anomaly." Because this exact schema — no more, no less — is what all three engines receive, the comparison between them measures purely how well each *reasons*, not whether one engine simply had access to better information than another.

---

## 7. Non-Functional Requirements

- **Reliability:** the scheduler must be self-evidently alive at all times (via heartbeat logs); a missed tick should be detectable, not silent.
- **Performance:** dashboard API responses should return in under 500ms under realistic data volumes (a few hundred endpoints, tens of thousands of check rows) — achieved primarily through the indexing strategy defined in 05_SCHEMA.md.
- **Security:** passwords are hashed with bcrypt before storage; JWTs are short-lived; endpoint auth headers/secrets are never written to application logs.
- **Explainability (project-specific NFR, tied directly to the PRD's research objective):** every diagnosis, from every engine, must include a genuine explanation string — a bare classification label with no accompanying reasoning is treated as a contract violation, not an acceptable shortcut, because explainability is one of the explicit evaluation dimensions the mentor asked for.
- **Portability:** local development must work entirely without any cloud dependency except the LLM API calls themselves — no hard-coded dependency on a specific hosting provider's proprietary features.
- **Reproducibility (research-specific NFR):** given the same labeled dataset version and the same engine, the evaluation harness should produce consistent, comparable results across separate runs (accounting for the inherent, bounded non-determinism of LLM-based engines).

---

## 8. Coding Conventions

- **Verbose, explicit code is mandatory project-wide.** See 08_RULES.md, Section 1, for the full rule set with concrete before/after examples — this Tech Spec defers to that document as the canonical style reference rather than duplicating it here.
- **File naming:** `camelCase.js` for backend files, `PascalCase.jsx` for React components.
- **Database access:** exclusively through Prisma's generated client; raw SQL is permitted only for aggregate queries Prisma cannot express cleanly, and any such query must include a comment explaining exactly what it computes and why Prisma's query builder wasn't sufficient.
- **Environment configuration:** all secrets and environment-specific values (`DATABASE_URL`, `GROQ_API_KEY`, `JWT_SECRET`, SMTP credentials) are loaded via `.env` locally and platform-provided secrets in deployment — never hardcoded, never committed.
- **Error handling:** every async operation that can fail (HTTP requests, database calls, external API calls) is wrapped in an explicit `try/catch`, with errors logged with enough context (which endpoint, which incident, which engine) to be debuggable after the fact.

---

## 9. Third-Party Dependencies (Complete List)

| Package | Language | Purpose |
|---|---|---|
| express | Node.js | HTTP server framework |
| prisma / @prisma/client | Node.js | Database ORM and migrations |
| pg | Node.js | PostgreSQL driver, used internally by Prisma |
| bcrypt | Node.js | Password hashing |
| jsonwebtoken | Node.js | JWT creation and verification |
| node-cron | Node.js | Scheduled health-check ticks |
| axios | Node.js | Outbound HTTP requests (health checks, LLM API calls, Python service calls) |
| nodemailer | Node.js | Email alerting via SMTP |
| dotenv | Node.js | Environment variable loading |
| react, react-dom | Frontend | UI library |
| react-router-dom | Frontend | Client-side routing |
| recharts | Frontend | Response-time and evaluation charts |
| vite | Frontend | Build tool and dev server |
| fastapi, uvicorn | Python | Serving Engine 2 and Engine 3 as HTTP inference services |
| scikit-learn | Python | Engine 2 model training, cross-validation, and evaluation metrics |
| xgboost | Python | Engine 2 candidate model (gradient-boosted trees) |
| pandas, numpy | Python | Feature engineering and dataset manipulation |
| transformers | Python | Loading and fine-tuning the base LLM for Engine 3 |
| peft | Python | LoRA/QLoRA parameter-efficient fine-tuning |
| bitsandbytes | Python | Quantization support enabling QLoRA on limited GPU memory |
| joblib | Python | Serializing/deserializing the trained Engine 2 model |

---

## 10. Explicitly Deferred Technical Decisions

Some technical choices are intentionally left open until the relevant implementation phase is reached, rather than decided prematurely:

- **Exact base model for Engine 3 fine-tuning** — to be selected in Phase 5 (Implementation Plan) based on what fits within available free-tier GPU memory at that time.
- **Final choice among Decision Tree / Random Forest / XGBoost for Engine 2** — to be decided empirically in Phase 4 based on actual cross-validation results on the labeled dataset, not assumed in advance.
- **Exact hosting provider for the deployed system** — Render/Railway are the default assumption, with AWS EC2 as an explicit stretch goal; the final choice depends on time available in Phase 7.
