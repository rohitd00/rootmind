# RootMind — To-Do / Tracker (Agent-Facing, In-Depth Edition)

**Document Version:** 3.0 (Deep Expansion)
**Companion to:** 06_IMPLEMENTATION_PLAN.md, 08_RULES.md
**Audience:** An AI coding agent (or the developer) working through this project task by task. This document is deliberately more granular than the Implementation Plan — where that document explains phases and reasoning, this document breaks each phase down into individually checkable, sequential tasks, each with enough detail that no additional interpretation should be needed to start working on it.

**Status legend:** `[ ]` Not started · `[~]` In progress · `[x]` Done · `[!]` Blocked (note the blocker in the Running Notes section at the bottom)

**How to use this tracker:** work top to bottom, phase by phase. Do not mark a task `[x]` until it has been manually verified working, per its own "Verify" note where one is given — not merely "the code was written." If a task is blocked (e.g., waiting on a decision, a credential, or a prior task), mark it `[!]` and log why in the Running Notes section rather than skipping past it silently.

---

## Phase 0 — Project Setup

### Backend Skeleton
- [ ] Run `npm init -y` in the backend project directory
- [ ] Install core dependencies: `npm install express dotenv cors`
- [ ] Install dev dependencies: `npm install --save-dev nodemon`
- [ ] Create folder structure: `/src/routes`, `/src/controllers`, `/src/services`, `/src/jobs`, `/src/middleware`
- [ ] Write `/src/index.js` — Express app setup, `cors()` middleware enabled for the local frontend origin, a `GET /health` route returning `{ status: "ok" }`
- [ ] Add an `npm run dev` script using `nodemon` for auto-restart during development
- [ ] **Verify:** `npm run dev` starts without errors; `curl http://localhost:3000/health` (or the configured port) returns the expected JSON

### Database and Prisma
- [ ] Provision a local PostgreSQL instance (Docker container or native install)
- [ ] Install Prisma: `npm install prisma --save-dev` and `npm install @prisma/client`
- [ ] Run `npx prisma init` to scaffold `schema.prisma` and `.env`
- [ ] Set `DATABASE_URL` in `.env` to point at the local PostgreSQL instance
- [ ] Write the initial Prisma schema: `User`, `Endpoint`, `Check` models only (per 05_SCHEMA.md — `Incident`, `Diagnosis`, and evaluation tables are added in later phases when first needed)
- [ ] Run `npx prisma migrate dev --name init`
- [ ] **Verify:** `npx prisma studio` shows all three tables with the correct columns and zero rows

### Frontend Skeleton
- [ ] Scaffold a Vite + React project (`npm create vite@latest`)
- [ ] Install `react-router-dom`
- [ ] Set up route placeholders for: `/login`, `/dashboard`, `/endpoints/:id`, `/incidents`, `/incidents/:id`, `/evaluation`, `/settings` — each rendering just a page title for now
- [ ] Confirm the frontend can call the backend's `GET /health` route and display its response (proves connectivity and CORS configuration)
- [ ] **Verify:** navigating between all placeholder routes works with no console errors; the health-check call succeeds and renders

### Version Control and Documentation
- [ ] Initialize Git repository (`git init`)
- [ ] Write `.gitignore`: `node_modules/`, `.env`, `dist/`, `__pycache__/`, `*.db`, any local dump files
- [ ] Write base `README.md`: project one-paragraph summary, backend setup/run instructions, frontend setup/run instructions, pointer to the full documentation set
- [ ] Make the initial commit and push to a remote repository

**Phase 0 exit check:** Can a teammate clone the repo fresh, follow the README exactly, and get both the backend and frontend running locally within a few minutes, with the frontend successfully talking to the backend? If yes, Phase 0 is complete.

---

## Phase 1 — Core Monitoring Engine

### Authentication
- [ ] Install `bcrypt` and `jsonwebtoken`
- [ ] `POST /api/auth/register` — validate email format server-side, validate password length (min 8 chars), check for existing account by email, hash password with bcrypt (cost factor ≥10), create `User` row with `alertEmailEnabled: true` by default
- [ ] `POST /api/auth/login` — look up user by email, compare password with `bcrypt.compare`, issue JWT (7-day expiry, signed with `JWT_SECRET` from `.env`) on success, generic "Incorrect email or password" error on any failure (never reveal which part was wrong)
- [ ] Auth middleware — read `Authorization: Bearer <token>` header, verify JWT, attach `req.user.id`; return 401 on missing/invalid/expired token
- [ ] Apply auth middleware to every route except `/api/auth/*`
- [ ] Frontend: Login/Register combined page — email/password fields, mode toggle, inline validation (per App Flow Flow A.1 and Design doc Section 5.8), JWT storage in `localStorage` on success, redirect to `/dashboard`
- [ ] Frontend: global 401 handler — on any API response with status 401, clear stored JWT and redirect to `/login` with a "session expired" message
- [ ] **Verify:** register a new account, confirm the password is hashed (not plaintext) in the database via `prisma studio`; log in with correct and incorrect credentials and confirm both paths behave as specified

### Endpoint CRUD
- [ ] `POST /api/endpoints` — validate all fields server-side (name/URL required, method against allow-list, status code range, timeout bounds); perform the one-time synchronous test ping; if it fails and `forceSave` is not set, return the structured failure response (no DB write); if it succeeds or `forceSave: true`, create the `Endpoint` row with `nextCheckAt: now()`
- [ ] `GET /api/endpoints` — list all `isActive: true` endpoints for `req.user.id`, with computed current status, 24h uptime %, and average response time per endpoint
- [ ] `GET /api/endpoints/:id` — single endpoint detail, scoped to the requesting user (return 404 if it belongs to someone else, not 403 — avoids confirming the ID's existence to unauthorized requesters)
- [ ] `PUT /api/endpoints/:id` — update configuration, same validation as create, no re-test-ping
- [ ] `DELETE /api/endpoints/:id` — soft-delete (`isActive: false`)
- [ ] `GET /api/endpoints/:id/checks` — paginated check history, most recent first, with an optional `range` query param (`24h` default, `7d` alternate) for the graph view
- [ ] Frontend: Add/Edit Endpoint form (modal or page) — all fields from App Flow Flow A.2, inline validation, the "Fix and Retry" / "Save Anyway" dual-button flow on test-ping failure
- [ ] Frontend: Dashboard endpoint list — sorted Down-then-Degraded-then-Up, status badges, empty state for zero endpoints, 30-second polling with Page Visibility API pause/resume
- [ ] Frontend: Endpoint Detail page — header with status and "since" duration, response-time graph (24h/7d toggle), paginated recent-checks table, edit/pause/delete actions with appropriate confirmation dialogs
- [ ] **Verify:** add an endpoint pointing at a real reachable URL and confirm it saves normally; add one pointing at an unreachable URL and confirm the Fix-and-Retry/Save-Anyway flow triggers correctly; confirm soft-deleted endpoints disappear from the Dashboard but their historical data remains queryable directly via Prisma Studio

### Health Check Scheduler
- [ ] Install `node-cron` and `axios`
- [ ] Set up a cron job ticking every 30 seconds
- [ ] Implement the due-endpoints query: `isActive: true AND nextCheckAt <= now()`
- [ ] Implement per-endpoint check execution using `axios`, wrapped in try/catch, using `Promise.allSettled` across all due endpoints in a tick (never `Promise.all`)
- [ ] Implement error classification: `TIMEOUT`, `CONNECTION_REFUSED`, `DNS_FAILURE`, `SSL_ERROR` (from caught exceptions), `NON_2XX` (from a received-but-mismatched response)
- [ ] Write a `Check` row for every attempt, success or failure, including `responseSnippet`/`responseHeaders` only on failure (truncate snippet to 500 characters)
- [ ] Update `nextCheckAt = now() + checkIntervalSec` after every check attempt, regardless of outcome
- [ ] Log a heartbeat line per tick: endpoints checked, failures, any skipped-due-to-error count
- [ ] Build 2–3 small local test services (simple Express apps) that can be configured to always succeed, always fail with a specific status, or hang past a timeout, for reliable scheduler testing
- [ ] **Verify:** point RootMind at the local test services; confirm checks run on the configured interval, confirm all five error types can each be triggered and correctly classified, confirm the heartbeat log appears every 30 seconds without gaps

### Incident Manager
- [ ] Add `Incident` model to Prisma schema, migrate
- [ ] Define `FAILURE_THRESHOLD` (default 2) and `FLAP_COOLDOWN_SECONDS` (default 120) as named constants with comments explaining their purpose
- [ ] Implement `handleFailedCheck(endpointId, checkId)` exactly per App Flow Part B.2's pseudocode
- [ ] Implement `handleSuccessfulCheck(endpointId, checkId)` exactly per App Flow Part B.2's pseudocode, including the flap-cooldown check
- [ ] Wire both functions into the scheduler's per-check outcome handling
- [ ] `GET /api/incidents` — list, filterable by `status` query param, scoped to the requesting user's endpoints
- [ ] `GET /api/incidents/:id` — detail, including the specific `Check` rows involved
- [ ] Frontend: Incidents List page — status filter (Open/Resolved default Open), incident cards per Design doc Section 5.4, empty states for each filter
- [ ] Frontend: Incident Detail page (first pass, no Diagnosis Panel yet) — timeline header, raw check-evidence table
- [ ] **Verify:** using the local test services, trigger 2 consecutive failures and confirm an incident opens; trigger a success immediately after and confirm the flap-cooldown correctly delays resolution per the worked example in App Flow Part B.2; confirm a sustained success eventually resolves the incident with a correctly calculated `durationSec`

### Alerting
- [ ] Install `nodemailer`
- [ ] Configure SMTP credentials in `.env` (a free provider or a personal account with an app-specific password)
- [ ] Implement incident-open email sending, triggered from the Incident Manager's incident-creation path
- [ ] Implement incident-resolve email sending, triggered from the Incident Manager's resolution path
- [ ] `PUT /api/settings` — update `alertEmailEnabled`
- [ ] Frontend: Settings page — email display (read-only), alert toggle, immediate save on toggle with revert-on-failure behavior
- [ ] **Verify:** trigger a real incident open/resolve cycle with alerting enabled and confirm both emails arrive with correct content; disable alerting and confirm no emails are sent on a subsequent cycle

**Phase 1 exit check (full end-to-end scenario, per 06_IMPLEMENTATION_PLAN.md Section 3.3):** register → log in → add a test endpoint → watch it get checked and show "Up" → break it → watch an incident open on the Dashboard and Incidents List, with an email if enabled → fix it → watch the incident resolve automatically, with a resolution email if enabled. If this entire sequence works without any manual database intervention, Phase 1 is genuinely done.

---

## Phase 2 — Diagnosis Engine 1 (Prompt-Based LLM)

- [ ] Implement `buildDiagnosisContext(incidentId)` — gathers failure signal, last N checks (`recentHistory`), correlated failures across other endpoints in the same time window, and 30-day historical failure rate for the endpoint; returns the exact shape from Tech Spec Section 6
- [ ] Write the system prompt: state the failure taxonomy explicitly, require JSON-only output matching the shared contract, include 3–5 few-shot examples spanning distinct failure types
- [ ] Install and configure the Groq API client (or chosen provider); store the API key in `.env`
- [ ] Implement the API call, response parsing, and JSON schema validation against the shared diagnosis contract shape
- [ ] Implement retry-on-malformed-response logic (max 2 retries) with a final fallback to an "unavailable" result if retries are exhausted
- [ ] Add `Diagnosis` model to Prisma schema, migrate
- [ ] Implement the Diagnosis Orchestrator per App Flow Part B.3's pseudocode, triggered asynchronously on new incident creation
- [ ] `POST /api/incidents/:id/diagnose` — accepts an `engine` query param (`prompt` is the only valid value at this point), manually re-runs diagnosis
- [ ] Frontend: Diagnosis Panel component per Design doc Section 5.5 — engine tag, ranked-causes bars, explanation paragraph, suggested-next-step callout, all three states (`completed`/`pending`/`unavailable`) handled distinctly
- [ ] Wire the Diagnosis Panel into the Incident Detail page, with 5–10 second polling while any diagnosis is `pending`
- [ ] Manual validation: deliberately trigger at least 4 distinct known failure types via the local test services (timeout, connection-refused, rate-limit/429, correlated multi-endpoint failure) and manually review each diagnosis for plausibility
- [ ] **Verify:** every new incident from the live scheduler automatically receives a diagnosis within a reasonable time; manually re-triggering via the API/UI works; temporarily breaking the API key or provider connectivity correctly results in an "unavailable" diagnosis state rather than a crash

**Phase 2 exit check:** does every new incident automatically get a plain-English diagnosis, visible on the Incident Detail page, within a few seconds to low tens of seconds?

---

## Phase 3 — Labeled Dataset Generation

- [ ] Finalize the failure taxonomy list (confirm against Schema doc Section 6; note any refinements discovered during Phase 2's manual testing)
- [ ] Build 4–6 controllable test services simulating: timeout, connection-refused, DNS-failure-equivalent, SSL-error-equivalent, rate-limiting (429), and a slow-climbing-latency (resource-exhaustion-style) pattern
- [ ] Run chaos-engineering sessions: trigger each failure type at least 15–20 times under live monitoring, capturing real `Check`/`Incident` data as ground-truth-labeled examples
- [ ] Write a synthetic data generation script (Python) with a fixed random seed, producing realistic varied context payloads labeled by generation target category
- [ ] (Optional) Use Engine 1 to help draft additional synthetic examples/explanations; manually review and correct every one before inclusion
- [ ] Export the final dataset to `/data/labeled_incidents_v1.json`, following the exact structure in Schema doc Section 8, including `generationMethod` per example
- [ ] Write `DATASET.md`: total count, per-category breakdown, chaos-vs-synthetic split, generation methodology
- [ ] Commit the dataset file and `DATASET.md` to version control
- [ ] **Verify:** every category in the taxonomy has a reasonable number of examples (flag and prioritize any category with fewer than ~10 examples for additional generation if time allows)

**Phase 3 exit check:** a versioned, documented dataset file exists with a reasonably balanced spread across all failure categories.

---

## Phase 4 — Diagnosis Engine 2 (Custom ML Model)

- [ ] Write the feature-engineering script (Python, pandas) converting context payloads to fixed feature vectors; document the exact feature list and encoding in `FEATURES.md`
- [ ] Split the Phase 3 dataset into train/test (or set up k-fold cross-validation); **save this exact split** for reuse in Phase 5 and Phase 6
- [ ] Train Decision Tree, Random Forest, and XGBoost classifiers on the training split
- [ ] Evaluate all three on the held-out test split: accuracy, per-class precision/recall/F1, confusion matrix
- [ ] Select the best-performing model based on actual results (not assumption); save it via `joblib` or XGBoost's native format
- [ ] Build the FastAPI inference service (`/diagnose` route), reusing the exact same feature-engineering logic from the first task, returning the shared diagnosis contract shape with a feature-importance-based explanation
- [ ] Wire the Node.js Diagnosis Orchestrator to call this service for `engine=ml`
- [ ] Record training/evaluation results (accuracy, F1, confusion matrix) to a results file for later comparison
- [ ] **Verify:** `POST /api/incidents/:id/diagnose?engine=ml` against a real incident returns a valid, contract-shaped diagnosis; confirm the model's live inference-time features match training-time feature engineering exactly (no drift)

**Phase 4 exit check:** the ML engine is callable via the same API surface as Engine 1, with recorded, defensible evaluation metrics.

---

## Phase 5 — Diagnosis Engine 3 (Fine-Tuned LLM)

- [ ] Reformat the Phase 3 dataset as instruction-following training examples (context in, full reasoning + diagnosis out)
- [ ] Select the base model (7–8B parameter class, open-weight) based on actual available free-tier GPU memory at this point in the project
- [ ] Set up the LoRA/QLoRA fine-tuning pipeline (`transformers`, `peft`, `bitsandbytes`)
- [ ] Run fine-tuning with regular checkpointing (to survive free-tier session interruptions); save final adapter weights
- [ ] Evaluate on the **exact same held-out test split** used for Engine 2 in Phase 4
- [ ] Deploy the fine-tuned model behind a FastAPI inference service matching the shared diagnosis contract
- [ ] Wire the Node.js Diagnosis Orchestrator to call this service for `engine=finetuned`
- [ ] Record training/evaluation results for later comparison
- [ ] **Verify:** `POST /api/incidents/:id/diagnose?engine=finetuned` returns a valid diagnosis; evaluation metrics are recorded and comparable (same test split) to Engine 2's

**Phase 5 exit check:** the fine-tuned engine is callable via the same API surface as the other two, with recorded evaluation metrics on the shared held-out test split.

**Risk reminder:** if compute/time runs short here, evaluate and report the best available checkpoint honestly rather than skipping this phase silently — Engines 1 and 2 alone already satisfy the core project requirement if this phase must be scoped down.

---

## Phase 6 — Comparative Evaluation Harness

- [ ] Add `EvaluationRun` and `EvaluationResult` models to Prisma schema, migrate
- [ ] Implement the evaluation harness per App Flow Part B.4's pseudocode — iterate every labeled example × all three engines, record top-1/top-3 correctness and latency, score engine failures as incorrect (not excluded)
- [ ] `POST /api/evaluation/run` — trigger a full run
- [ ] `GET /api/evaluation/results` — retrieve latest or a specified run's aggregated results
- [ ] Frontend: Evaluation Results page — summary table, latency/cost bar charts, expandable per-engine confusion matrices, CSV/JSON export
- [ ] Document the cost-per-1,000-diagnoses estimation methodology per engine (API pricing for Engine 1; approximate compute cost for Engines 2 and 3)
- [ ] Run the full evaluation at least once against the complete labeled dataset
- [ ] Export and archive the results
- [ ] **Verify:** the comparison table's numbers are internally consistent (e.g., top-3 accuracy is never lower than top-1 accuracy for the same engine); spot-check a handful of individual `EvaluationResult` rows against the raw dataset to confirm correctness scoring logic is working as intended

**Phase 6 exit check:** a completed evaluation run exists with results for all three engines, viewable and exportable.

---

## Phase 7 — Polish, Deployment, Documentation

- [ ] Deploy backend + database (Render/Railway, or EC2 as stretch)
- [ ] Deploy frontend (same platform or static hosting)
- [ ] Deploy Python inference services (Engine 2, Engine 3)
- [ ] Walk every screen against the Design doc's Section 9 consistency checklist; fix any deviations found
- [ ] Write the final project report (PRD problem/goals + Tech Spec architecture + Schema data model + Phase 6 results as the empirical core)
- [ ] Script and rehearse the viva demo sequence using the controllable local test services (never relying on real third-party API behavior for demo reliability)
- [ ] (Stretch) Draft the comparative-study paper using Phase 6 results
- [ ] **Verify:** the deployed system is reachable at a stable URL and the full demo sequence runs start-to-finish without manual database intervention

**Phase 7 exit check:** deployed, checklist-passed, report written, demo rehearsed at least once successfully.

---

## Running Notes / Blockers Log

*(Append dated entries here as work progresses — decisions made, issues hit, anything that changes the plan from what's written above. Never leave a `[!]` blocked task without a corresponding note here explaining what it's waiting on.)*

- `YYYY-MM-DD` — note here
