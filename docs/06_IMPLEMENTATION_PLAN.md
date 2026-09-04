# RootMind — Implementation Plan (In-Depth Edition)

**Document Version:** 3.0 (Deep Expansion)
**Companion to:** 01_PRD.md, 02_TECH_SPEC.md, 03_APP_FLOW.md, 04_DESIGN.md, 05_SCHEMA.md
**Purpose:** This document is the complete build roadmap for RootMind — not just a list of phases, but a full explanation of *why* the work is sequenced this way, exactly what "done" means for each phase, what could go wrong at each step and how to respond, and how the phases connect to each other so that time invested early isn't wasted or duplicated later. Anyone picking up this project mid-way through — a teammate, a future version of the author, or an AI coding agent — should be able to read this document and know exactly where to pick up and why the prior work was structured the way it was.

---

## 1. Sequencing Philosophy — The Reasoning Behind the Order

Before listing the phases themselves, it's worth being explicit about the four principles that determined their order, since the order itself is one of the most important decisions in this entire plan:

### 1.1 Protect a Complete, Demoable Fallback as Early as Possible

The single biggest risk to any student project spanning a full semester is running out of time before anything is genuinely finished. RootMind's plan is deliberately structured so that **Phase 1 alone constitutes a complete, working, gradeable product** — a real uptime monitor with scheduling, incident detection, and a functioning dashboard. If every later phase (AI diagnosis, the three-engine comparison, the evaluation harness) somehow fell through due to time constraints, Phase 1's output is still a legitimate, demoable major project. Every subsequent phase is additive value layered on top of a foundation that already stands on its own.

### 1.2 Build Shared Infrastructure Once, Not Three Times

The three AI diagnosis engines (prompt-based, custom ML, fine-tuned LLM) all consume the exact same labeled dataset and the exact same context-payload contract. Rather than building data-generation and evaluation tooling separately for each engine as it's built, this plan generates the labeled dataset **once**, in a dedicated phase (Phase 3), positioned deliberately *before* Engine 2 and Engine 3 work begins — since both of those engines depend on it, and building it once, properly, is strictly cheaper in total effort than building it piecemeal three separate times.

### 1.3 Sequence From Least to Most Resource-Dependent

Engine 1 (prompt-based) requires no training data and no GPU — just an API key and prompt engineering. Engine 2 (custom ML) requires labeled data and modest local/cloud compute. Engine 3 (fine-tuned LLM) requires labeled data *and* GPU compute, and is the most operationally fragile of the three (free-tier GPU sessions can be interrupted, fine-tuning runs can fail partway through and need re-running). Building in this order means the most resource-constrained, highest-risk work happens last, when its scope can be adjusted downward if time is tight — rather than first, where an early setback could derail the whole semester's planning.

### 1.4 Leave Room for the Unplanned

This plan intentionally does not pack every available week with new feature work. Time is set aside for polish, deployment friction (which is almost always underestimated), and demo rehearsal — because a technically complete system that fails to demo smoothly during the actual viva undermines the entire semester's work in a way that's disproportionate to the actual engineering gap it represents.

---

## 2. Phase 0 — Project Setup and Environment

### 2.1 Objective

Establish a working, empty-but-connected full-stack skeleton: a backend that runs and responds, a frontend that loads and can talk to the backend, a database that's reachable, and version control in place — before any real feature logic is written. This phase exists specifically to front-load all the "getting the pieces to talk to each other" friction (dependency installation, environment variable wiring, CORS configuration, database connection strings) so that Phase 1's actual feature work isn't interrupted by infrastructure surprises partway through.

### 2.2 Detailed Task List

1. **Backend initialization:**
   - Run `npm init` in a new backend directory; install `express`, `dotenv`, and `nodemon` as a dev dependency.
   - Create the folder structure: `/src/routes`, `/src/controllers`, `/src/services`, `/src/jobs`, `/src/middleware`, `/prisma`.
   - Write a minimal `/src/index.js` that starts an Express server and exposes a single `GET /health` route returning `{ status: "ok" }` — this becomes the very first thing tested to confirm the server runs at all.
2. **Database and Prisma setup:**
   - Provision a PostgreSQL instance (local, via a Docker container or a native local install, for day-to-day development; a free-tier hosted instance, e.g., on Render or Railway, for eventual shared/deployed use).
   - Install `prisma` and `@prisma/client`; run `npx prisma init` to scaffold the `schema.prisma` file and `.env` template.
   - Write the initial schema covering just `User`, `Endpoint`, and `Check` (per 05_SCHEMA.md) — `Incident`, `Diagnosis`, and the evaluation tables are deliberately deferred to the phases that actually need them, since introducing them now would mean staring at empty, unused tables for weeks and risk the schema drifting from this document if changes happen before they're actually exercised by real code.
   - Run the first migration (`npx prisma migrate dev --name init`) and confirm, via `npx prisma studio` or a direct `psql` inspection, that the tables exist with the expected columns.
3. **Frontend initialization:**
   - Scaffold a new Vite + React project.
   - Set up `react-router-dom` with placeholder route components for every screen in the App Flow's screen inventory (Login/Register, Dashboard, Endpoint Detail, Incidents, Incident Detail, Evaluation, Settings) — even if each placeholder just renders its own title text for now. This gives the whole app's navigation shape early, so later phases are filling in already-connected pages rather than also having to wire up new routes as they go.
   - Confirm the frontend can successfully call the backend's `GET /health` route and render its response, proving the two halves of the stack can actually communicate (including confirming CORS is configured correctly on the backend for local development).
4. **Version control and documentation:**
   - Initialize a Git repository; write a `.gitignore` covering `node_modules/`, `.env`, `dist/`, `__pycache__/`, and any local database dump files.
   - Write a base `README.md` covering: what the project is (a one-paragraph summary drawn from the PRD's executive summary), how to run the backend locally, how to run the frontend locally, and where the rest of the documentation (this file and its companions) lives.
   - Make an initial commit and push to a remote repository.

### 2.3 Definition of Done

- `npm run dev` (or equivalent) starts the backend and it responds successfully to `GET /health`.
- The frontend loads in a browser, and navigating between the placeholder routes works without errors.
- The frontend can successfully fetch and display the backend's health-check response, proving connectivity end-to-end.
- `npx prisma studio` shows the `User`, `Endpoint`, and `Check` tables, correctly structured, with zero rows (since no real data has been created yet).
- The Git repository is initialized, the initial commit is pushed, and the README accurately describes how to get the project running from scratch.

### 2.4 Common Pitfalls at This Stage

- **CORS misconfiguration:** a very common early stumbling block — the frontend (running on, e.g., `localhost:5173`) calling the backend (running on `localhost:3000`) will be blocked by the browser unless the backend explicitly allows that origin via the `cors` middleware package. Resolve this in Phase 0, not partway through Phase 1 when it becomes a confusing distraction from feature logic.
- **`.env` values not loading:** confirm `dotenv.config()` is called before any code that reads `process.env` values, and confirm the `.env` file is actually present locally (since it's gitignored and won't exist on a fresh clone without manually creating it) — worth noting this explicitly in the README's setup instructions to save future-you or a teammate confusion.

---

## 3. Phase 1 — Core Monitoring Engine

### 3.1 Objective

Build the complete, standalone uptime-monitoring product described in the PRD's "safe MVP" — authentication, endpoint management, the scheduled health-check loop, incident detection with the consecutive-failure and flap-cooldown logic, a working dashboard, and basic email alerting. By the end of this phase, RootMind is already a real, useful, demoable tool, with no AI involved yet at all.

### 3.2 Detailed Task Breakdown

#### 3.2.1 Authentication

- Implement `POST /api/auth/register`: validate email format and password strength server-side, check for existing accounts, hash the password with `bcrypt` (cost factor 10+), create the `User` row.
- Implement `POST /api/auth/login`: look up the user by email, compare the submitted password against the stored hash using `bcrypt.compare`, issue a JWT (using `jsonwebtoken`, signed with a secret from environment variables, with a 7-day expiry) on success.
- Implement an Express middleware function that reads the `Authorization: Bearer <token>` header, verifies the JWT, and attaches the decoded user ID to `req.user` — applied to every route except `/api/auth/*`.
- **Frontend:** build the combined Login/Register page per App Flow Flow A.1 and Design doc Section 5.8's form styling — including the specific inline-error behaviors described in that flow (the "email already exists" link to switch modes, the deliberately non-specific "incorrect email or password" message).

#### 3.2.2 Endpoint CRUD

- Implement `POST /api/endpoints`, including the one-time synchronous test-ping logic described in App Flow Flow A.2, and the `forceSave` flag handling for the "Save Anyway" path.
- Implement `GET /api/endpoints` (list, scoped to `req.user.id`), `GET /api/endpoints/:id` (detail), `PUT /api/endpoints/:id` (update), and `DELETE /api/endpoints/:id` (soft-delete, setting `isActive = false`).
- **Frontend:** build the Add/Edit Endpoint form (per Design doc Section 5.8), the Dashboard's endpoint list view (per Design doc Section 5.2, including the sort-order behavior from App Flow Flow A.3), and the Endpoint Detail page's configuration/pause/delete actions (App Flow Flow A.4).

#### 3.2.3 Health Check Scheduler

- Set up a `node-cron` job ticking every 30 seconds.
- Implement the core due-endpoints query (`Endpoint(nextCheckAt)` index from the Schema doc directly supports this).
- Implement the per-endpoint check execution using `axios`, including the full error classification logic (`TIMEOUT`, `CONNECTION_REFUSED`, `DNS_FAILURE`, `SSL_ERROR`, `NON_2XX`) exactly as specified in App Flow Part B.1's pseudocode — using `Promise.allSettled` (not `Promise.all`) across endpoints in a single tick, for the isolation reasons explained there.
- Implement the heartbeat log line at the end of each tick.
- **Testing approach for this specific piece:** stand up two or three simple local test services (a plain Express app or two) that can be configured to always succeed, always fail with a specific status code, or hang past a timeout — using these as controlled, repeatable test targets is far more reliable for verifying scheduler correctness than depending on real, unpredictable third-party APIs during development.

#### 3.2.4 Incident Manager

- Add the `Incident` model to the Prisma schema and migrate.
- Implement `handleFailedCheck` and `handleSuccessfulCheck` exactly per App Flow Part B.2's pseudocode, including the consecutive-failure threshold (default 2) and the flap-cooldown window (default 120 seconds) as named, easily-adjustable constants (per the Rules document's "no unexplained magic numbers" requirement).
- Implement `GET /api/incidents` (list, filterable by `status`) and `GET /api/incidents/:id` (detail).
- **Frontend:** build the Incidents List page (Design doc Section 5.4, App Flow Flow A.6) and a first version of the Incident Detail page showing just the raw timeline and check evidence (the Diagnosis Panel itself is deferred to Phase 2, since there's no diagnosis to show yet).

#### 3.2.5 Alerting

- Set up `nodemailer` with SMTP credentials (a free provider or a personal Gmail account with an app-specific password is sufficient for development and demo purposes).
- Implement incident-open and incident-resolve email sending, wired into the Incident Manager's transitions.
- Implement `PUT /api/settings` for the `alertEmailEnabled` toggle, and the corresponding Settings page (App Flow Flow A.9).

### 3.3 Definition of Done — the Phase 1 Exit Test

Phase 1 is complete when the following end-to-end scenario works correctly, without manual database intervention:

1. A new user registers and logs in.
2. They add a real endpoint (or one of the local test services from Section 3.2.3) via the Add Endpoint form.
3. Within roughly 30–60 seconds, the Dashboard shows the endpoint's status as "Up" (or "Pending first check" briefly beforehand).
4. The endpoint is deliberately broken (e.g., the local test service is reconfigured to fail, or stopped).
5. Within two check intervals, the Dashboard's status badge changes to "Down," and a new entry appears on the Incidents List.
6. If email alerting is enabled, an incident-open email arrives.
7. The endpoint is fixed/restarted.
8. Within one more check interval (respecting the flap-cooldown logic), the incident resolves, the Dashboard returns to "Up," and (if enabled) a resolution email arrives.

If this entire sequence works reliably and repeatably, Phase 1 is genuinely done — not just "the code is written," but verified working end-to-end.

### 3.4 Why This Is a Natural Mentor Checkpoint

This phase's completion is a strong, well-defined point to schedule a mentor progress review, since the system is functionally whole at this point — a real monitoring product, even without any AI yet — and demonstrating it working end-to-end is a low-risk, high-confidence checkpoint before moving into the more experimental AI phases.

---

## 4. Phase 2 — Diagnosis Engine 1: Prompt-Based LLM

### 4.1 Objective

Layer the first AI-powered root-cause diagnosis engine onto the now-complete monitoring core, making "AI-powered" a real, working claim rather than an aspiration. This phase deliberately targets the lowest-effort, no-training-data-required engine first, so the product's headline differentiating feature exists as early as possible in the timeline.

### 4.2 Detailed Task Breakdown

1. **Context payload construction:** implement `buildDiagnosisContext(incidentId)` exactly per the Tech Spec's schema (Section 6) and the Schema doc's field definitions — gathering the failure signal, recent history, correlated failures across other endpoints, and historical failure rate. This function is written once and reused, unchanged, by every future engine (a core architectural commitment from the Tech Spec, worth re-emphasizing here since it's easy to accidentally special-case this function while focused only on Engine 1's needs).
2. **Prompt design:** write the system prompt for the Groq API call, including:
   - An explicit statement of the failure taxonomy (the canonical list from Schema doc Section 6).
   - Clear instructions to return only a JSON object matching the shared diagnosis contract (Tech Spec Section 4.4) — ranked causes with confidence scores, an explanation, and a suggested next step.
   - 3–5 few-shot examples covering distinct failure types (a timeout with a latency-creep pattern, a sudden connection-refused failure, a correlated multi-endpoint outage, a rate-limiting 429 response) so the model has concrete pattern examples to generalize from, not just abstract category names.
3. **API integration:** implement the Groq API call using `axios`, including request construction, response parsing, and JSON schema validation of the parsed response against the expected shape.
4. **Retry and fallback logic:** if the response fails schema validation (malformed or unexpected JSON), retry up to 2 times with the same input before giving up; on final failure, return a result that the Diagnosis Orchestrator will record as `status: "unavailable"`.
5. **Diagnosis Orchestrator and data model:** add the `Diagnosis` model to Prisma and migrate; implement the orchestration logic exactly per App Flow Part B.3's pseudocode, wired to fire asynchronously whenever the Incident Manager creates a new incident.
6. **Manual re-trigger endpoint:** implement `POST /api/incidents/:id/diagnose`, accepting an `engine` query parameter (even though only `"prompt"` is a valid value at this point in the build — the route's shape is already future-proofed for Engines 2 and 3).
7. **Frontend — Diagnosis Panel:** build the full Diagnosis Panel component per Design doc Section 5.5, including all of its documented states (completed, pending, unavailable), and wire it into the Incident Detail page.
8. **Manual validation:** deliberately break a test endpoint in at least 4 distinct, known ways (a hard timeout, a connection-refused failure, a 429 rate-limit response, and a scenario with correlated failures across two endpoints simultaneously) and manually review whether each resulting diagnosis is reasonable — this is the closest thing to "testing" a subjective AI output has at this stage, and doing it deliberately across a spread of failure types (not just re-testing the same scenario repeatedly) gives real signal on whether the prompt is generalizing well or is overfit to one pattern.

### 4.3 Definition of Done — the Phase 2 Exit Test

Every new incident, created by the real monitoring engine from Phase 1, automatically receives a diagnosis within a reasonable time window (a few seconds to low tens of seconds, dominated by LLM API latency) — visible on the Incident Detail page without any manual action required. Manually re-triggering diagnosis via the UI also works correctly. At least the 4 manually-tested failure scenarios from Section 4.2's final task produce diagnoses that a reasonable person would judge as plausible given the underlying data.

### 4.4 Risk Note Specific to This Phase

LLM API responses are inherently non-deterministic and occasionally malformed even with careful prompting. The retry-then-fallback design (Section 4.2, task 4) is what keeps this an acceptable, well-handled risk rather than a source of visible bugs — verify during manual testing that the fallback path (a temporarily broken or rate-limited API) is exercised at least once and behaves as expected (a clean "Diagnosis unavailable" state, not a crashed request or an unhandled exception).

---

## 5. Phase 3 — Labeled Dataset Generation

### 5.1 Objective

Produce the single, versioned, labeled dataset that both Engine 2 (Phase 4) and Engine 3 (Phase 5) depend on, and that also serves as the shared benchmark for the Phase 6 comparative evaluation. This phase is positioned deliberately before either of those engine-building phases, precisely because building this dataset once, carefully, is cheaper and more consistent than building it twice, separately, under time pressure later.

### 5.2 Detailed Task Breakdown

1. **Finalize the failure taxonomy** — confirm the canonical list from Schema doc Section 6 is final (or note any additions/refinements discovered during Phase 2's manual testing) before dataset labeling begins, since changing the taxonomy after labeling has started means re-labeling already-collected examples.
2. **Build controllable test services** — 4 to 6 small, purpose-built Express applications (or simple scripts), each capable of being deliberately configured to produce one specific failure type on demand: a service that sleeps past any configured timeout, one that returns 429 on command, one that can be made to refuse connections, one that simulates a slow-climbing latency curve to mimic resource exhaustion, and so on.
3. **Run chaos-engineering sessions** — with RootMind's own monitoring actively watching these controllable test services, deliberately trigger each failure type multiple times (aim for at least 15–20 repetitions per category, more if time allows, to give the eventual ML/fine-tuning training data reasonable per-class representation), capturing the real `Check`/`Incident` data RootMind records as ground-truth-labeled examples, since the injected cause is known with certainty.
4. **Generate synthetic data to scale up volume** — write a script that programmatically generates realistic-looking context payloads (varying latency curves, status codes, correlated-failure patterns) with a known, deterministic random seed (per the Rules document's reproducibility requirement), labeled by the category the generation logic targeted. This is what allows the dataset to reach a meaningfully larger size than chaos sessions alone could practically produce within the available time.
5. **Optional LLM-assisted example drafting** — use Engine 1 (already built in Phase 2) to help draft additional synthetic examples and plausible explanations, with every such example manually reviewed and corrected before inclusion — a distillation-style approach that can meaningfully speed up dataset creation, as long as the manual review step is never skipped, since unreviewed LLM-generated "ground truth" would undermine the entire premise of a labeled evaluation set.
6. **Export and version the dataset** — write the final dataset as `/data/labeled_incidents_v1.json`, structured exactly per the format shown in the Schema doc's Section 8, including the `generationMethod` field (`"chaos"` vs. `"synthetic"`) on every example.
7. **Document the dataset** — write a short `DATASET.md` file covering: total example count, per-category breakdown, the chaos-vs-synthetic split, and the generation methodology in enough detail that the eventual report/paper's methodology section can largely be drawn from it directly.

### 5.3 Definition of Done

A versioned dataset file exists, containing a reasonably balanced spread of examples across every failure category in the taxonomy, with clear provenance (chaos vs. synthetic) per example, documented in `DATASET.md`, and committed to version control.

### 5.4 A Note on "Reasonably Balanced"

Perfect class balance is not required or even necessarily realistic (some failure types, like DNS failures, may be harder to safely and repeatably simulate than others, like timeouts) — but a category with only 1–2 examples total will be nearly impossible for Engine 2 to learn anything meaningful about and will produce statistically unreliable accuracy figures for that category in Phase 6's evaluation. If time allows, prioritize additional chaos/synthetic generation specifically for whichever categories end up thinnest, rather than spreading remaining effort evenly.

---

## 6. Phase 4 — Diagnosis Engine 2: Custom ML Model

### 6.1 Objective

Build the second of the three mentor-specified diagnosis approaches — a classical, fully explainable machine learning classifier trained on the Phase 3 dataset — and integrate it into RootMind behind the same shared diagnosis contract Engine 1 already implements.

### 6.2 Detailed Task Breakdown

1. **Feature engineering** (Python, `pandas`): write a script that converts each labeled example's context payload into a fixed-length feature vector — candidate features include: the latency slope across `recentHistory` (a simple linear fit or a first-vs-last-value delta), the raw `errorType` (one-hot encoded), the `statusCode` (bucketed, e.g., into 4xx/5xx/none), the count of `correlatedFailures`, and `historicalFailureRate`. Document the exact feature list and encoding scheme in a short `FEATURES.md`, since this exact same feature-extraction logic must be reused, unchanged, at inference time (Section 6.2, task 5) — any drift between training-time and inference-time feature engineering is a classic, easy-to-introduce bug that silently corrupts model predictions.
2. **Train/test split:** split the Phase 3 dataset (e.g., 80/20, or using k-fold cross-validation given the likely modest overall size) — critically, using the **same held-out test split** that will later be reused for Engine 3 (Phase 5) and referenced in Phase 6's evaluation, so all three engines are ultimately judged on identical unseen data, per the Rules document's research-integrity requirement.
3. **Train and compare candidate models** (`scikit-learn`, `xgboost`): train a Decision Tree, a Random Forest, and an XGBoost classifier on the same training split; evaluate each on the held-out test split using accuracy, per-class precision/recall/F1, and a confusion matrix.
4. **Select the best-performing model** based on these results — not by assumption — and export it (`joblib` for scikit-learn models, or XGBoost's native save format).
5. **Build the inference service:** a small FastAPI application exposing `POST /diagnose`, which accepts the same context-payload JSON shape as the other engines, runs it through the exact same feature-engineering logic from task 1, loads the saved model, and returns a response in the shared diagnosis contract shape — with the explanation text templated from the model's feature importances (e.g., "The response time increased steadily before failure and no other endpoints were affected, which most strongly suggests resource exhaustion on this specific service.").
6. **Wire into RootMind:** update the Node.js Diagnosis Orchestrator to call this new service when `engine=ml` is requested, exactly the same way it calls Engine 1's in-process logic — from the Orchestrator's point of view, this is just another HTTP-reachable engine implementing the same contract.
7. **Record training results:** save the accuracy/F1/confusion-matrix numbers from task 3 to a results file — this becomes direct input to the Phase 6 comparison and the eventual report, so it should be captured and preserved now rather than needing to be re-run later purely to regenerate a number that was already computed once.

### 6.3 Definition of Done

`POST /api/incidents/:id/diagnose?engine=ml`, called against a real incident from the live system, returns a valid, contract-shaped diagnosis. The training results (accuracy, F1, confusion matrix) are recorded and match, on the held-out test split, what's reported in the results file from task 7.

### 6.4 A Note on Choosing "The Best" Model Honestly

Section 6.2's task 4 deliberately says the best-performing model should be selected *based on the actual results*, not assumed in advance (the Tech Spec explicitly defers this choice for the same reason — see Tech Spec Section 10). If, in practice, a simple Decision Tree performs comparably to XGBoost on this particular dataset (which is entirely possible on a modest-sized, engineered feature set), that is itself a legitimate and reportable finding — there's no obligation to force the "fanciest" model to win if the data doesn't support it, and reporting an honest, sometimes-surprising result is more valuable for the research angle than reporting a foregone conclusion.

---

## 7. Phase 5 — Diagnosis Engine 3: Fine-Tuned LLM

### 7.1 Objective

Build the third and most resource-intensive diagnosis approach: a small, open-weight LLM specialized on the Phase 3 dataset via parameter-efficient fine-tuning, aiming to combine natural-language reasoning quality with domain specialization.

### 7.2 Detailed Task Breakdown

1. **Reformat the dataset for fine-tuning:** convert the Phase 3 labeled examples (currently formatted as raw context-payload-plus-label pairs) into instruction-following training examples — each one framed as "given this context [payload], provide a diagnosis" paired with a target output matching the shared diagnosis contract's shape, including a well-written explanation string (not just the bare label), so the fine-tuned model learns to produce full, explanatory outputs, not just classifications.
2. **Select the base model:** choose a small, open-weight model in the 7–8B parameter range (the exact choice deferred until this phase, per Tech Spec Section 10, based on what fits comfortably within the memory constraints of whatever free-tier GPU is actually available at this point — Colab and Kaggle's free-tier GPU offerings can vary, so this decision is made with current, real constraints in hand rather than assumed months in advance).
3. **Set up the fine-tuning pipeline** (`transformers`, `peft`, `bitsandbytes`): configure LoRA (or QLoRA, if memory constraints require the additional quantization) targeting the base model's attention layers, with a modest number of trainable adapter parameters appropriate for the dataset size from Phase 3 (a small dataset calls for a conservative training configuration — few epochs, a moderate learning rate — to avoid overfitting on what is likely a relatively small number of examples compared to typical fine-tuning corpora).
4. **Run training**, checkpointing regularly (free-tier GPU sessions can be time-limited or interrupted, so frequent checkpointing is a practical necessity, not just good practice) and saving the final LoRA adapter weights.
5. **Evaluate on the exact same held-out test split** used for Engine 2 in Phase 4 — this reuse of the identical split (not a fresh, independently-drawn one) is what makes the eventual three-way comparison in Phase 6 valid.
6. **Deploy the fine-tuned model behind an inference endpoint:** a FastAPI service (following the same pattern as Engine 2's) that loads the base model plus the fine-tuned adapter weights and exposes a `/diagnose` route matching the shared contract — hosted either on the same free-tier compute used for training (if it supports persistent hosting) or a separate lightweight always-on service, depending on what's practically available.
7. **Wire into RootMind:** update the Diagnosis Orchestrator to support `engine=finetuned`, identically to how Engine 2 was integrated in Phase 4.

### 7.3 Definition of Done

`POST /api/incidents/:id/diagnose?engine=finetuned` returns a valid, contract-shaped diagnosis from the fine-tuned model. Evaluation metrics (top-1/top-3 accuracy) on the shared held-out test split are recorded for direct comparison against Engine 2's Phase 4 results.

### 7.4 Explicit Risk Management for This Phase

This is the phase most likely to be squeezed by time or compute constraints, and the plan treats it accordingly:

- **If GPU compute or time runs out mid-training:** the most recent checkpoint can be evaluated and reported as-is, with a note in the report that further training was constrained by available compute — a documented, honest limitation is entirely acceptable in an academic context and is far preferable to silently skipping this engine with no explanation.
- **If there isn't time to build Engine 3 at all:** per the PRD's risk table and the Rules document's Section 6, Engines 1 and 2 alone already constitute a legitimate, complete two-way comparison satisfying the core "AI-powered diagnosis" requirement — the mentor conversation flagged in the PRD's Open Questions (whether partial results are acceptable for the mid-term) should be revisited early if this risk starts to materialize, rather than waiting until it's already happened.

---

## 8. Phase 6 — Comparative Evaluation Harness and Results

### 8.1 Objective

Build the module that actually produces the project's core research output: a rigorous, reproducible, side-by-side comparison of all three diagnosis engines across accuracy, latency, cost, and explainability — this is the concrete deliverable the mentor's comparative-paper suggestion depends on.

### 8.2 Detailed Task Breakdown

1. Add the `EvaluationRun` and `EvaluationResult` models to the Prisma schema (per Schema doc Section 3) and migrate.
2. Implement the evaluation harness logic exactly per App Flow Part B.4's pseudocode — iterating every labeled example against all three engines, recording top-1/top-3 correctness and latency for each, and correctly scoring engine failures as incorrect rather than excluding them (per the reasoning given in that section).
3. Implement `POST /api/evaluation/run` and `GET /api/evaluation/results`.
4. Build the Evaluation Results frontend page per Design doc's charting guidelines and App Flow Flow A.10 — the summary table, latency/cost comparison charts, expandable confusion matrices, and CSV/JSON export.
5. **Estimate cost per 1,000 diagnoses per engine:** for Engine 1, based on the LLM provider's published token pricing multiplied by average tokens per call; for Engine 2, based on approximate compute cost of running the lightweight inference service (likely near-zero on a small free/low-cost instance, worth stating honestly as such); for Engine 3, based on the hosting cost of the self-hosted inference endpoint — document the estimation methodology for each, since "cost" means something different for a pay-per-call API versus self-hosted infrastructure, and a fair comparison needs to be explicit about that difference rather than presenting three numbers as if they were computed identically.
6. **Run the full evaluation** at least once against the complete labeled dataset, and export the results.

### 8.3 Definition of Done

A completed `EvaluationRun` exists with results for all three engines across the full labeled dataset, viewable on the Evaluation Results page, exportable as CSV/JSON, with the cost-estimation methodology for each engine documented clearly enough to defend in a viva if questioned.

---

## 9. Phase 7 — Polish, Deployment, and Documentation

### 9.1 Objective

Take the technically-complete system from Phases 1–6 and turn it into something that demos reliably, looks finished, and is backed by documentation ready for submission — closing out the semester's work in a controlled, rehearsed way rather than scrambling right up to the viva date.

### 9.2 Detailed Task Breakdown

1. **Deployment:** deploy the Node.js backend, PostgreSQL database, and React frontend to Render or Railway (per the Tech Spec's default choice); deploy the Python inference services (Engines 2 and 3) similarly. If time allows, additionally deploy some portion of the system to AWS EC2 as the explicitly-optional stretch goal for cloud-deployment skill-building — but only after the primary deployment is solid, never at the expense of it.
2. **UI polish pass:** walk through every screen against the Design doc's Section 9 consistency checklist, fixing any spacing, color, or responsive-behavior inconsistencies found.
3. **Write the final project report,** structured around: the PRD's problem statement and goals, the Tech Spec's architecture, the Schema's data model, and — as the centerpiece — the Phase 6 evaluation results, presented as the project's core empirical contribution.
4. **Prepare and rehearse a scripted demo sequence** for the viva: a fixed, reliable sequence (e.g., "add a pre-prepared test endpoint → trigger a known failure via the chaos test services from Phase 3 → show the incident appearing on the dashboard → show the diagnosis panel populate → manually re-run diagnosis with a second engine for live comparison → show the Evaluation Results page's aggregate comparison") — deliberately built around controllable test services rather than real, unpredictable third-party APIs, so the demo's timing and outcome are dependable regardless of what's happening on the public internet on demo day.
5. **(Stretch) Draft the comparative-study paper,** using the Phase 6 results as the empirical core, following whatever format/venue guidance comes back from the mentor open question in the PRD.

### 9.3 Definition of Done

The system is deployed and reachable at a stable URL, every screen passes the Design doc's consistency checklist, the final report is written and internally consistent with the actual, as-built system (not the originally-planned one, if anything diverged along the way), and the demo sequence has been rehearsed at least once, start to finish, without needing any manual database intervention or unplanned improvisation.

---

## 10. Summary Timeline

| Phase | Weeks (Reference) | Focus | Exit Condition |
|---|---|---|---|
| 0 | 1 | Project setup | Full-stack skeleton runs and is connected |
| 1 | 2–4 | Core monitoring engine | End-to-end monitor → incident → resolve cycle works reliably |
| 2 | 5–6 | Engine 1 — Prompt-based LLM diagnosis | Every incident gets an automatic, plausible diagnosis |
| 3 | 7–8 | Labeled dataset generation | Versioned, documented, reasonably balanced dataset exists |
| 4 | 9–10 | Engine 2 — Custom ML model | Trained, evaluated, integrated ML engine callable via API |
| 5 | 11–12 | Engine 3 — Fine-tuned LLM | Fine-tuned, evaluated, integrated LLM engine callable via API |
| 6 | 13 | Comparative evaluation harness | Full three-engine comparison results, exportable |
| 7 | 14 | Polish, deployment, documentation | Deployed, checklist-passed, report written, demo rehearsed |

This timeline is a planning reference, not a rigid contract — align the actual calendar dates against your semester's real mentor checkpoints, the August 20–31 synopsis submission window, and the mid-term/final evaluation dates, adjusting individual phase lengths as needed while preserving the overall sequencing logic explained in Section 1, especially the principle of protecting Phases 1 and 2 above all else if time pressure forces a scope cut later in the semester.
