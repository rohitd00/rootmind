# RootMind — Product Requirements Document (PRD)

**Project Name:** RootMind
**Project Type:** B.Tech CSE Major Project — 7th Semester
**Domain:** API Health Monitoring + AI-Powered Root-Cause Diagnosis
**Team Context:** Neural Nomads (project/PBL team)
**Document Version:** 2.0 (Expanded)
**Status:** Mentor-approved; synopsis submission window August 20–31
**Author's Role:** Project lead, backend-leaning full-stack developer (Node.js, Express, PostgreSQL)

---

## 1. Executive Summary

RootMind is a full-stack, AI-augmented API health monitoring and uptime tracking system. At its foundation, it behaves like any standard uptime monitor: it periodically checks a set of user-registered API endpoints, records whether each is healthy, and alerts the user when something breaks. What differentiates RootMind — and what forms the academic core of this major project — is a second layer sitting on top of that foundation: an AI-powered root-cause diagnosis engine that looks at *why* an endpoint failed, not just *that* it failed, and explains the likely cause in plain English.

The project has two intertwined objectives:

1. **An engineering objective** — build a genuinely working, deployable, demoable monitoring product (RootMind) with real value on its own merits.
2. **A research objective**, set explicitly by the mentor — implement and rigorously compare three fundamentally different approaches to generating that diagnosis:
   - **Approach 1:** Prompt-based diagnosis using an off-the-shelf LLM accessed via API token (no training).
   - **Approach 2:** A custom-trained classical ML model, trained from labeled failure data collected/generated specifically for this project.
   - **Approach 3:** A fine-tuned LLM, specialized on the same labeled data using parameter-efficient fine-tuning.

   The mentor has indicated that a detailed comparative paper across these three approaches — evaluated on accuracy, latency, cost, and explainability — is a viable and encouraged outcome of this project, published after the system and evaluation are complete.

This PRD defines what RootMind must do, for whom, why, and how success will be measured — for both the product and the research components.

---

## 2. Background and Motivation

### 2.1 The Problem in Detail

Every non-trivial modern application depends on a web of APIs — internal microservices, third-party payment gateways, authentication providers, data services, and more. When one of these dependencies degrades or fails, the standard tooling available to most developers (uptime pings, basic status dashboards, generic alerting) tells them **that** something is wrong but almost nothing about **why**.

In practice, engineers respond to an outage by manually:
- Opening logs and searching around the failure timestamp
- Checking recent deploys or config changes
- Pinging the failing service manually to see what error comes back
- Cross-referencing whether other services failed at the same time (suggesting shared infrastructure)
- Drawing on personal experience/pattern-matching ("this looks like the DB connection pool again")

This is slow, repetitive, and heavily dependent on tribal knowledge that lives in individual engineers' heads rather than in the tooling itself. Many failure causes are, in fact, highly recognizable from signal patterns alone — a steadily climbing latency curve followed by a timeout very often means resource exhaustion; a sudden hard failure across multiple unrelated endpoints at once usually means shared infrastructure (DNS, load balancer, network) rather than an application-level bug; a 429 status code is definitionally rate limiting. RootMind's founding hypothesis is that these patterns are learnable — either through careful prompting of a general-purpose LLM, through a trained classifier, or through a fine-tuned specialized model — and that automating this first-pass diagnosis meaningfully shortens the time between "something broke" and "I know roughly where to look."

### 2.2 Why This Is a Suitable Major Project

- It combines **systems engineering** (a real-time scheduler, database design, API design, alerting) with **applied AI/ML** (prompt engineering, classical ML training and evaluation, LLM fine-tuning) — giving the project depth across both halves of a CSE curriculum.
- It naturally produces **quantifiable, comparable results** (accuracy, latency, cost across three methods), which satisfies the mentor's request for a paper-worthy contribution rather than a purely descriptive build.
- It is scoped so that a **safe, complete MVP exists early** (the core monitoring engine, Phase 1 of the Implementation Plan) — meaning the project is gradeable and demoable even if the more ambitious AI comparison work runs into time constraints later in the semester.
- It reuses and extends skills already developed in prior/parallel work (Node.js/Express/PostgreSQL stack, Groq/LLaMA API familiarity from other projects), reducing ramp-up time and letting more of the semester go toward the novel research component.

### 2.3 Why Existing Tools Don't Solve This

Commercial uptime tools (UptimeRobot, Pingdom, StatusCake, Better Uptime, and similar) are mature at the "detect and alert" half of this problem, but none of them ship a root-cause reasoning layer that explains *why* a specific failure happened in plain language, and none of them are built as a testbed for comparing different AI methodologies against each other on this task — which is precisely the gap RootMind is positioned to explore academically, even though it does not aim to compete with these tools commercially.

---

## 3. Goals

### 3.1 Primary Goals

1. Build a reliable, scheduled API health monitoring engine that checks registered endpoints and accurately records their health history over time.
2. Detect failures and performance degradation with a low false-positive rate, avoiding noisy or "flapping" incident alerts.
3. Automatically generate a human-readable root-cause diagnosis for every detected incident, including ranked probable causes, a confidence estimate, and a suggested next diagnostic step.
4. Implement all three mentor-specified AI approaches (prompt-based, custom ML, fine-tuned LLM) behind a single shared interface, so they can be run interchangeably against the same incidents.
5. Build a rigorous, reproducible evaluation harness that compares all three approaches on the same labeled dataset, across accuracy, latency, cost, and explainability.
6. Produce a report (and, as a stretch goal, a submittable paper) presenting these comparative findings.

### 3.2 Secondary Goals

1. Provide a clean, minimal, and genuinely usable dashboard — good enough to demo confidently in a viva and credible enough to serve as a portfolio piece independent of the academic angle.
2. Keep the entire system deployable on free-tier or low-cost infrastructure, since this is a self-funded student project.
3. Write code that is clear and teachable (see the dedicated Rules document) so that teammates, mentors, and future-you can read and extend it without friction.
4. Where time allows, deploy part of the system to AWS (e.g., EC2) as a deliberate opportunity to build cloud deployment experience alongside the core coursework.

### 3.3 Non-Goals (Explicitly Out of Scope)

- **Multi-tenant SaaS features** — no organizations, teams, billing, or per-seat account management. RootMind is single-user-account-scoped for this project.
- **Enterprise-scale monitoring** — no requirement to support thousands of endpoints, multi-region probing, or high-availability clustering of the monitoring engine itself.
- **Full incident management workflows** — no on-call scheduling, escalation policies, or paging integrations (PagerDuty, Opsgenie, etc.). Email alerting is sufficient.
- **Auto-remediation** — RootMind diagnoses problems; it never attempts to automatically fix, restart, or reconfigure anything. This is a deliberate boundary to keep scope (and risk) contained.
- **Non-REST protocol support** — GraphQL and gRPC endpoint monitoring are not required for the MVP; the system assumes standard HTTP/HTTPS REST-style endpoints unless explicitly revisited with the mentor.
- **Commercial launch or public user acquisition** — this is an academic project first; any real-world usability is a beneficial side effect, not a goal to plan around.

---

## 4. Target Users and Personas

| Persona | Description | Primary Needs | How RootMind Serves Them |
|---|---|---|---|
| **Solo developer / small team engineer** | Runs a handful of personal or small-team services (side projects, small startups, coursework infra) and wants lightweight monitoring without enterprise tooling overhead | Fast setup, minimal configuration, clear plain-English alerts they can act on immediately | Simple endpoint registration, calm dashboard, automatic diagnosis reduces manual debugging time |
| **Academic evaluator (mentor / project panel)** | Assesses the project for technical correctness, engineering rigor, and research novelty during checkpoints, mid-terms, and the final viva | A working, demonstrable system; clearly reasoned methodology; honest, reproducible comparative results | Structured PRD/Tech Spec/Evaluation results; a scripted, reliable live demo; transparent metrics rather than cherry-picked claims |
| **Future recruiter / portfolio reviewer** | Looks at this project post-graduation as evidence of full-stack + applied-AI capability | Clean, readable code; sensible architecture; a project that demonstrates independent technical judgment, not just following a tutorial | A genuinely original system combining backend engineering with three distinct AI methodologies, documented professionally |
| **Project teammate (Neural Nomads)** | May contribute to or maintain parts of the codebase during the project's lifecycle | Clear documentation, consistent and readable code style, well-defined module boundaries | The full documentation set (this PRD plus the Tech Spec, Schema, App Flow, Rules) exists specifically so teammates can onboard without needing verbal explanation |

---

## 5. Core Features (Detailed)

### 5.1 Endpoint Management

- Users can add, view, edit, pause, and delete monitored API endpoints.
- Each endpoint's configuration includes: display name, URL, HTTP method (GET/POST/etc.), expected status code(s) for a "healthy" response, custom headers (for auth tokens or API keys the target endpoint requires), check interval (e.g., every 1/5/15 minutes), and timeout threshold (in milliseconds) beyond which a check is considered failed even without an explicit error.
- On adding a new endpoint, the system performs a one-time test ping immediately, both to catch obvious misconfiguration early and to give the user immediate feedback rather than waiting for the next scheduled tick.
- Deleting an endpoint is a **soft delete** (marked inactive rather than removed from the database), preserving historical check/incident data for both the user's own reference and for the integrity of any evaluation results derived from real incidents.

### 5.2 Health Checking Engine

- A background scheduler runs continuously, checking each active endpoint according to its configured interval.
- Every check attempt — successful or not — is recorded, including: exact timestamp, HTTP status code returned (if any), response time in milliseconds, a success/failure flag, and, specifically on failure, an error classification (timeout, connection refused, DNS failure, SSL/TLS error, non-matching status code) plus a short snippet of the response body and headers for later diagnostic use.
- The engine distinguishes between **hard failures** (the request could not complete at all, or returned a non-matching status code) and **soft degradation** (the request succeeded, but response time exceeded a configurable latency threshold) — both are meaningful signals for diagnosis, not just binary up/down.

### 5.3 Incident Detection and Lifecycle

- A single failed check does not immediately create a visible "incident" — this avoids false alarms from one-off network blips. Instead, an incident is opened only once a configurable number of **consecutive** failures occurs (default: 2).
- Once open, an incident aggregates all relevant context: the triggering checks, the endpoint's recent history leading up to the failure, and (once implemented) the diagnosis output.
- The incident automatically closes/resolves the moment a subsequent scheduled check for that endpoint succeeds again, with the total duration calculated and stored.
- A **flap-cooldown** mechanism prevents rapidly alternating success/failure results from generating a storm of separate incidents; flapping within the cooldown window is treated as a continuation of the same incident rather than a new one.

### 5.4 AI-Powered Root-Cause Diagnosis (Core Differentiator)

For every opened incident, RootMind assembles a structured "diagnosis context payload" containing everything a human debugging the issue would want to know at a glance: the failure signal itself (status code, error type, response time), the recent history of checks leading up to the failure (to spot a gradual latency creep versus a sudden hard failure), whether other monitored endpoints failed in the same time window (suggesting shared infrastructure trouble rather than an isolated problem), and this endpoint's historical failure rate (to distinguish a first-time anomaly from a recurring known issue).

This exact same context payload is then handed to whichever "diagnosis engine" is active, and each engine returns: a ranked list of probable causes with confidence scores, a plain-English explanation suitable for a non-specialist to read, and a suggested next diagnostic step a human could take to confirm the cause.

The three diagnosis engines, all built to this same shared contract so they are directly interchangeable and comparable:

- **Engine 1 — Prompt-Based (API Token):** A carefully engineered system prompt (including a defined failure taxonomy and a handful of few-shot examples) is sent to an off-the-shelf LLM (Groq/LLaMA, chosen for speed and prior familiarity) along with the context payload; the model's structured JSON response is parsed into the shared diagnosis format. No training data or model ownership required — fastest to build, but ongoing per-call cost and dependence on an external provider.
- **Engine 2 — Custom-Trained ML Model:** A classical machine learning classifier (candidates: Decision Tree, Random Forest, XGBoost — selected empirically) trained on engineered features derived from labeled failure examples collected specifically for this project. Produces fast, fully explainable (via feature importances) predictions with no ongoing external API dependency, at the cost of needing labeled training data upfront and being limited to failure types represented in that data.
- **Engine 3 — Fine-Tuned LLM:** A small open-weight LLM (7–8B parameter class) fine-tuned via parameter-efficient methods (LoRA/QLoRA) on the same labeled dataset, reformatted as instruction-following examples. Aims to combine the natural-language reasoning quality of an LLM with domain specialization, while removing per-call dependency on an external API once deployed — at the cost of needing GPU compute for training and a more complex pipeline overall.

### 5.5 Comparative Evaluation Module (Research Core)

- A dedicated, versioned labeled dataset of failure scenarios — combining controlled "chaos engineering" sessions (deliberately breaking test services in known ways to capture real, ground-truth-labeled signal patterns) and synthetic data generation (scripted variation of realistic context payloads at larger scale) — underpins fair evaluation of all three engines.
- An evaluation harness runs every engine against every example in a held-out test split of this dataset and computes, per engine: top-1 accuracy (did the top-ranked cause match ground truth exactly), top-3 accuracy (was the correct cause anywhere in the top three ranked causes), per-class confusion matrices (which failure types get mistaken for which), average latency per diagnosis, and an estimated cost per 1,000 diagnoses (factoring in API pricing for Engine 1, and compute/hosting cost estimates for Engines 2 and 3).
- Results are stored in the database (kept fully separate from live production incident data, to protect both the integrity of real monitoring history and the reproducibility of research results) and are exportable as CSV/JSON for direct use in the final report or paper.

### 5.6 Dashboard (Minimal, Functional UI)

- **Endpoint List / Dashboard Home:** every monitored endpoint at a glance — current status (Up/Down/Degraded), 24-hour uptime percentage, and average response time.
- **Endpoint Detail:** a focused view of one endpoint's health over time — a response-time graph, a recent-checks table, and a list of past incidents specific to that endpoint.
- **Incidents List:** every incident across all endpoints, filterable by status (open/resolved) and time range.
- **Incident Detail:** the full incident timeline plus the Diagnosis Panel — ranked causes with confidence bars, the plain-English explanation, the suggested next step, and which engine produced the result, with an option to manually re-run diagnosis using a different engine for side-by-side comparison during a live demo.
- **Evaluation Results:** the mentor/mentee-facing comparison view — accuracy tables, latency and cost charts, and confusion matrices per engine.
- **Settings:** account details and the alert-email toggle.

### 5.7 Alerting

- Email notifications (via SMTP/Nodemailer) are sent when an incident opens and again when it resolves, including the endpoint name, timestamps, and — if ready in time — a short summary of the diagnosis. This is intentionally kept simple; richer notification channels (SMS, Slack, PagerDuty-style paging) are explicitly out of scope for this project's timeline.

---

## 6. Detailed User Stories

1. **As a developer**, I want to register an API endpoint with a name, URL, method, and check interval, so that RootMind begins monitoring it automatically without further manual intervention.
2. **As a developer**, I want to receive an email the moment an endpoint goes down, so that I can respond before the outage affects users of my own application.
3. **As a developer**, when I open a failed incident, I want to immediately see a plausible, plain-English explanation of what likely caused it, so I don't have to start my investigation from zero every time.
4. **As a developer**, I want to see an endpoint's uptime percentage and historical incidents over time, so that I can identify endpoints that fail repeatedly and deserve deeper investigation or replacement.
5. **As a developer**, I want the option to manually re-run diagnosis with a different engine on the same incident, so I can sanity-check whether different methods agree.
6. **As the mentor/evaluator**, I want to see a clear, side-by-side comparison of accuracy, latency, and cost across all three AI approaches, so I can assess the project's research contribution on its merits rather than take claims at face value.
7. **As the mentor/evaluator**, I want every diagnosis — regardless of which engine produced it — to include a human-readable explanation and not just a bare label, so that I can judge whether the reasoning itself is sound, not just whether the final answer happened to be correct.
8. **As a teammate joining the project**, I want documentation detailed enough (this PRD, the Tech Spec, the Schema, and the Rules doc) that I can start contributing without needing a verbal walkthrough first.

---

## 7. Success Metrics

| Category | Metric | Target |
|---|---|---|
| Monitoring reliability | Missed scheduled checks under normal load | Zero (100% of due checks executed on schedule) |
| Incident detection quality | False positive incident rate | Under 5%, achieved via the consecutive-failure threshold and flap-cooldown |
| Diagnosis accuracy | Top-1 accuracy, best-performing engine, on held-out labeled test set | 70% or higher |
| Diagnosis accuracy | Top-3 accuracy, best-performing engine | 90% or higher |
| Dashboard performance | Page load time for endpoint list/detail views | Under 2 seconds |
| Demo reliability | System uptime and correctness during the scripted viva demo | 100% — no live failures during evaluation |
| Research rigor | All three engines evaluated on an identical held-out test split | Enforced by design (see Rules doc, Section 3) |
| Documentation completeness | PRD, Tech Spec, App Flow, Design, Schema, Implementation Plan, To-Do Tracker, and Rules docs all kept current with the actual system | Maintained throughout, not just written once upfront |

---

## 8. Constraints

- **Team size and time:** This is primarily an individually-led effort within a small student team (Neural Nomads), balanced against coursework, other ongoing projects, and placement/interview preparation happening concurrently this semester.
- **Budget:** Zero to near-zero budget — reliance on free-tier LLM APIs (Groq), free/low-cost hosting (Render/Railway, or AWS free-tier/EC2 as a stretch), and free GPU compute for fine-tuning (Google Colab, Kaggle Notebooks).
- **Compute:** No dedicated local GPU; Engine 3's fine-tuning must use parameter-efficient methods (LoRA/QLoRA) runnable within free-tier compute session limits.
- **Timeline:** Bounded by the semester calendar — synopsis submission window is August 20–31, with mentor checkpoints, a mid-term evaluation, and a final demo/viva to plan around (see Implementation Plan for the full phase-by-phase schedule).

---

## 9. Assumptions

- A mix of real public API endpoints and deliberately-controlled internal test endpoints will be used for both everyday development testing and for generating the labeled dataset used in evaluation.
- The chosen LLM API provider (Groq, or an equivalent free/low-cost provider) remains available and within free-tier limits for the duration of the project.
- A single-server, single-instance deployment is entirely sufficient for this project's grading and demo purposes — no horizontal scaling, load balancing, or multi-region deployment is expected or necessary.
- The mentor's expectation for the comparative paper is a rigorous, honestly-reported comparison rather than one engine being artificially made to "win" — the evaluation harness is designed accordingly (see Rules doc, Section 3, on research integrity).

---

## 10. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Insufficient labeled failure data to properly train/evaluate Engine 2 and Engine 3 | Medium | High — blocks two of the three required approaches | Dedicated Phase 3 in the Implementation Plan combines controlled chaos-engineering sessions with scripted synthetic data generation specifically to solve this before Engine 2/3 work begins |
| Fine-tuning (Engine 3) exceeds available time or free-tier compute limits | Medium-High | Medium — could leave the three-way comparison incomplete | Engine 3 is explicitly the most expendable phase under time pressure (see Rules doc, Section 6); Engines 1 and 2 alone already satisfy a legitimate two-way comparison and the core "AI-powered diagnosis" requirement |
| LLM API rate limits or downtime during evaluation harness runs | Low-Medium | Medium — could stall or corrupt evaluation results | Evaluation runs are batched, results are cached as they complete, and a free-tier-aware provider (Groq) with generous limits is used |
| Scope creep — trying to build the full dashboard, alerting, and all three engines simultaneously | Medium | High — risks missing every deadline instead of some | Implementation Plan enforces strict phase sequencing; Phase 1 (core monitoring) alone is a complete, demoable fallback if later phases slip |
| Concurrent workload (placement prep, other coursework/projects) reduces available weekly hours | Medium | Medium | Implementation Plan's phase order protects the highest-value, lowest-risk work (core monitoring, then Engine 1) earliest in the timeline |

---

## 11. Open Questions for the Mentor

- Should preliminary comparative results (even if only Engines 1 and 2 are complete) be included in the mid-term evaluation, or is the full three-way comparison expected only at the final viva?
- Is there a minimum expected size or diversity for the labeled dataset for the comparison to be considered methodologically sound for a potential paper submission?
- Should RootMind's scope remain strictly REST/HTTP endpoints, or would extending to GraphQL be viewed as a valuable (optional) enhancement?
- Is there a preferred or required target venue/format in mind for the comparative paper, in case formatting or evaluation-metric choices should be aligned to it from the start?

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **Endpoint** | A specific API URL registered with RootMind for periodic health checking |
| **Check** | A single health-check attempt against an endpoint at a point in time |
| **Incident** | An open period of confirmed failure for an endpoint, opened after consecutive failed checks and closed on recovery |
| **Diagnosis** | The output of an AI engine explaining the probable root cause of a given incident |
| **Engine** | One of the three interchangeable methods (prompt-based, custom ML, fine-tuned LLM) used to generate a diagnosis |
| **Context Payload** | The structured bundle of failure signals and history handed identically to every engine for diagnosis |
| **Flap / Flapping** | An endpoint rapidly alternating between success and failure, which the flap-cooldown mechanism prevents from generating excessive incidents |
| **Top-1 / Top-3 Accuracy** | Evaluation metrics measuring whether the correct cause was the engine's single top guess, or anywhere within its top three ranked guesses |
| **LoRA / QLoRA** | Parameter-efficient fine-tuning techniques used to specialize Engine 3's LLM without full retraining |

---

## 13. Relationship to Other Documents

This PRD defines *what* RootMind must do and *why*. It is deliberately implementation-agnostic. For *how* it will be built, refer to:
- **02_TECH_SPEC.md** — technology choices, system architecture, API design
- **03_APP_FLOW.md** — screen-by-screen and process-by-process user/system flows
- **04_DESIGN.md** — visual design system and UI guidelines (minimalist)
- **05_SCHEMA.md** — full database schema and rationale
- **06_IMPLEMENTATION_PLAN.md** — phased build sequence and timeline
- **07_TODO_TRACKER.md** — granular, checkable task list for execution
- **08_RULES.md** — binding engineering and research-integrity rules, including the verbose/humanized code style requirement
