# RootMind — Rules to Follow (Agent-Facing, In-Depth Edition)

**Document Version:** 3.0 (Deep Expansion)
**Companion to:** All other RootMind documents — this file is the binding constraint layer that governs how every other document's intentions get implemented in actual code.
**Audience:** Any AI coding agent or human developer working on this codebase.
**Status:** Non-negotiable unless explicitly overridden, in writing, by the project owner. If any instruction elsewhere seems to conflict with a rule here, this document wins unless the project owner has explicitly said otherwise for that specific case.

---

## 1. Code Style — Humanized, Verbose Code (Primary Rule, Fully Explained)

This is the single most important stylistic rule governing this entire project: **write code the way a careful, experienced human engineer would write it for a teammate to read six months from now — not the most compressed way the language happens to allow.**

### 1.1 The Underlying Principle

Compressed, "clever" code optimizes for the moment of writing. Verbose, explicit code optimizes for every moment *after* writing — reading, debugging, explaining in a viva, modifying under time pressure, and being understood by a mentor or teammate who didn't write it. Given that this codebase will be read far more often than it's written (by the author revisiting it weeks later, by a mentor evaluating it, by a teammate potentially extending it), the reading experience is what this rule optimizes for, even at the cost of a few extra lines.

### 1.2 No Unnecessary Shorthand Syntax

Avoid deeply nested or chained ternaries, avoid squeezing multiple logical operations into a single dense line just because the language permits it.

**Bad:**
```js
const status = c.success ? (c.responseTimeMs > 1000 ? 'degraded' : 'up') : 'down';
```

**Good:**
```js
let status;
if (check.success === false) {
  status = "down";
} else if (check.responseTimeMs > 1000) {
  status = "degraded";
} else {
  status = "up";
}
```

**Why this specific example matters beyond just style:** the "bad" version requires the reader to mentally parse two nested conditionals and reconstruct the logic backward from the ternary's structure. The "good" version reads top-to-bottom in the same order a person would naturally reason about the problem: first check if it failed outright, then check if it was slow, and only then default to healthy. This ordering-matches-reasoning property is worth more than the four saved lines.

### 1.3 Full, Descriptive Names — No Cryptic Abbreviations

Use complete, descriptive variable and function names throughout. Single-letter variables are acceptable only for trivial loop counters in very short loops (`i`, `j`), and even then, a descriptive name (`checkIndex`, `endpointIndex`) is preferred whenever the loop body does anything beyond the most trivial iteration.

**Bad:**
```js
function calc(e, cfg) {
  const r = e.rt > cfg.th;
  return r;
}
```

**Good:**
```js
function isResponseTimeDegraded(endpointCheck, endpointConfig) {
  const isOverLatencyThreshold = endpointCheck.responseTimeMs > endpointConfig.timeoutMs;
  return isOverLatencyThreshold;
}
```

**Why this matters even for "obvious" functions:** a function name and its parameters should tell most of the story before the reader even looks at the body. `calc(e, cfg)` tells the reader nothing — they must read the entire function body just to learn what it computes. `isResponseTimeDegraded(endpointCheck, endpointConfig)` tells the reader almost everything before they've read a single line inside it.

### 1.4 Avoid Excessive Method Chaining in a Single Expression

Break multi-step data transformations into named intermediate variables, so each step is independently inspectable — and, critically, independently debuggable with a `console.log` or breakpoint placed on any one line.

**Bad:**
```js
const result = data.filter(x => x.active).map(x => x.value).reduce((a, b) => a + b, 0);
```

**Good:**
```js
const activeItems = data.filter((item) => item.active === true);
const activeValues = activeItems.map((item) => item.value);
const totalValue = activeValues.reduce((sum, value) => sum + value, 0);
```

**Why this matters practically:** if the "bad" version produces a wrong result, the developer has to mentally re-derive which of the three chained operations is at fault, or awkwardly break the chain apart temporarily just to inspect an intermediate value. The "good" version already has each intermediate value sitting in its own named variable — a debugger or a quick console log at any line immediately shows exactly what that stage of the computation produced.

### 1.5 Avoid Clever One-Liners, Even When They "Work"

If a single line of code needs a comment to explain what it's doing, it is a strong signal that it should be several lines of self-explanatory code instead of one clever line plus a comment translating it.

**Bad:**
```js
// increments the failure streak, or resets it to 1 if the streak was broken
const newStreak = check.success ? 0 : (previousStreak * (previousSuccess ? 0 : 1)) + 1;
```

**Good:**
```js
let newConsecutiveFailureCount;
if (check.success === true) {
  newConsecutiveFailureCount = 0;
} else if (previousCheckWasAlsoAFailure === true) {
  newConsecutiveFailureCount = previousConsecutiveFailureCount + 1;
} else {
  newConsecutiveFailureCount = 1;
}
```

The "good" version needs no explanatory comment at all, because the code itself already reads as a description of the logic.

### 1.6 Prefer Explicit `if/else` Over Nested or Chained Ternaries

A single, simple ternary used for a genuinely trivial inline default is acceptable (e.g., `const label = isActive ? "Active" : "Paused";`). Nested ternaries, or ternaries chained together to express more than a single true/false branch, are never acceptable — convert them to an explicit `if/else if/else` block instead, as shown in Section 1.2's example.

### 1.7 Prefer `async/await` With Explicit `try/catch` Over Chained Promises

**Bad:**
```js
fetchEndpointConfig(endpointId)
  .then((config) => runHealthCheck(config))
  .then((result) => saveCheckResult(result))
  .catch((error) => console.log(error));
```

**Good:**
```js
async function performScheduledCheck(endpointId) {
  try {
    const endpointConfig = await fetchEndpointConfig(endpointId);
    const checkResult = await runHealthCheck(endpointConfig);
    await saveCheckResult(checkResult);
  } catch (error) {
    console.error(`Failed to complete scheduled check for endpoint ${endpointId}:`, error);
  }
}
```

**Why this matters beyond style preference:** the `async/await` version reads top-to-bottom exactly like a description of the steps taken, with the error-handling boundary (the `try/catch`) visually wrapping exactly the steps it covers. The chained-promise version requires the reader to trace where each `.then()` picks up the previous one's return value, and the single trailing `.catch()` gives no immediately obvious indication of which of the three steps actually failed — the `async/await` version's error message, by contrast, can (and should) include enough context to make that clear immediately.

### 1.8 Avoid Destructuring-Heavy One-Liners That Obscure Data Origin

In controller/route-handler code especially — the code a mentor or teammate is most likely to read directly to understand what an API endpoint does — prefer explicit property access over heavy destructuring, particularly for request bodies with several optional fields.

**Bad:**
```js
const { name, url, method = "GET", expectedStatus = 200, checkIntervalSec = 60, timeoutMs = 5000, customHeaders } = req.body;
```

**Good:**
```js
const endpointName = req.body.name;
const endpointUrl = req.body.url;
const httpMethod = req.body.httpMethod || "GET";
const expectedStatusCode = req.body.expectedStatus || 200;
const checkIntervalInSeconds = req.body.checkIntervalSec || 60;
const timeoutInMilliseconds = req.body.timeoutMs || 5000;
const customRequestHeaders = req.body.customHeaders || null;
```

**Why this specific tradeoff is worth the extra lines:** the destructured version packs seven pieces of information (field name, source object, default value) into a single dense line that's easy to misread at a glance, especially as more fields are added over time. The explicit version makes each field's origin (`req.body`) and default value trivially clear on its own line, and is far easier to extend later — adding an eighth field means adding one new clear line, not carefully editing a already-long destructuring statement.

### 1.9 Comment the "Why," Not Just the "What"

Every non-trivial function should have a short comment above it explaining its purpose and any non-obvious design decisions — particularly *why* a specific number, threshold, or approach was chosen, since that reasoning is exactly what's lost if only the code itself survives.

**Example, drawn directly from this project:**
```js
// Consecutive-failure threshold before we open a real Incident.
// Set to 2 rather than 1 specifically to avoid opening an incident
// (and sending an alert email) for a single transient network blip —
// see App Flow doc, Part B.2, for the full reasoning and a worked
// flap-cooldown example.
const FAILURE_THRESHOLD = 2;
```

### 1.10 No Unexplained Magic Numbers

Every meaningful constant — the consecutive-failure threshold, the flap-cooldown window, retry limits, default timeout values, polling intervals — must be a named constant with a short explanatory comment, never a bare number scattered directly into the logic where it's used.

**Bad:**
```js
if (consecutiveFailures >= 2) {
  openIncident();
}
```

**Good:**
```js
// See comment on FAILURE_THRESHOLD's definition for the reasoning behind this value.
if (consecutiveFailures >= FAILURE_THRESHOLD) {
  openIncident();
}
```

### 1.11 This Rule Applies Across Every Language Used in the Project

The verbose/humanized code standard is not a Node.js/JavaScript-specific rule — it applies identically to the Python code written for Engines 2 and 3 (feature engineering, model training, fine-tuning scripts) and to any raw SQL written for aggregate queries Prisma can't express cleanly.

**Python-specific guidance:**
- Avoid dense list/dict comprehensions with multiple conditions or nested logic packed into one line — prefer a clear `for` loop with named intermediate variables when the transformation is non-trivial.
- Avoid unnecessary `lambda` functions where a small, named, regular function would be clearer — a named function can also carry a docstring explaining its purpose, which a `lambda` cannot.
- Avoid one-line multi-condition boolean expressions buried inside a larger statement — extract them into a named boolean variable first.

**Bad (Python):**
```python
features = [f(x) for x in data if x['success'] == False and x['statusCode'] is not None and x['statusCode'] >= 500]
```

**Good (Python):**
```python
def extract_feature_from_failed_check(check):
    """
    Builds one feature-vector row from a single failed check record.
    Only called for checks that already failed with a 5xx status code —
    see the calling loop for why other failure types are handled separately.
    """
    return build_feature_vector(check)


server_error_failures = []
for check in data:
    is_a_failure = check["success"] == False
    has_a_status_code = check["statusCode"] is not None
    is_a_server_error = has_a_status_code and check["statusCode"] >= 500
    if is_a_failure and is_a_server_error:
        server_error_failures.append(check)

features = [extract_feature_from_failed_check(check) for check in server_error_failures]
```

### 1.12 Rationale, Restated Plainly

This is a student project that will be read and evaluated by a mentor, and potentially extended by teammates from the Neural Nomads group. Code here needs to function as a teaching artifact — something that demonstrates and communicates understanding — not as a code-golf exercise in minimal keystrokes. Verbose, explicit code is also, very concretely, easier to defend line-by-line if questioned directly during a viva, since every line already reads as a clear statement of intent rather than requiring on-the-spot translation.

---

## 2. Architectural Rules (Expanded)

### 2.1 The Scheduler Must Never Block on Diagnosis

The health-check scheduler's core loop (App Flow Part B.1) must never wait on the Diagnosis Orchestrator (Part B.3) to complete before moving on. Diagnosis is always triggered as a fire-and-forget async call — a slow or entirely unreachable LLM API must never delay, throttle, or break the scheduler's ability to keep checking every other endpoint on time. **Concretely:** the call to `DiagnosisOrchestrator.diagnoseIncident(...)` from within `IncidentManager.handleFailedCheck` must not be `await`-ed in a way that blocks the scheduler tick's completion — it should be invoked and allowed to run to completion independently.

### 2.2 All Three Diagnosis Engines Must Implement the Identical Contract

No engine-specific field is ever allowed to leak into the shared `Diagnosis` data shape consumed by the frontend or the Evaluation Harness. If Engine 2's ML model happens to produce some additional internal metadata (e.g., raw feature-importance scores), that data can be used internally to *construct* the `explanation` string, but it is never exposed as an additional top-level field on the `Diagnosis` record itself. **Why this is enforced this strictly:** the entire validity of the three-way research comparison (the mentor's core ask) depends on every engine being evaluated against the exact same contract — any engine-specific extra fields would either need to be ignored (wasted work) or would create an unfair, inconsistent basis for comparison.

### 2.3 The Evaluation Harness Must Never Touch Live Production Tables

The evaluation harness (App Flow Part B.4) reads exclusively from the versioned labeled dataset file, and writes exclusively to `EvaluationRun`/`EvaluationResult`. It must never read from or write to `Incident`, `Check`, or the production-path `Diagnosis` rows. This is enforced at the code level by simply never importing or calling any production Incident/Check/Diagnosis query functions from within the evaluation harness module — the isolation should be structural (different modules, different database queries entirely), not just a convention that could accidentally be violated by a future edit.

### 2.4 Database Access Goes Through Prisma's Generated Client

Raw SQL is permitted only for aggregate queries that Prisma's query builder genuinely cannot express cleanly (e.g., certain complex window-function-based aggregations), and any such raw query must include a comment explaining exactly what it computes and, briefly, why Prisma's query builder wasn't sufficient for it.

### 2.5 Secrets Are Never Hardcoded and Never Committed

API keys (Groq/LLM provider), database credentials, JWT signing secrets, and SMTP credentials are always sourced from environment variables (`.env` locally, platform-provided secrets when deployed) and are never written directly into source code, never logged in plaintext (including in error logs — redact or omit sensitive values before logging an error object that might contain them), and never committed to version control.

---

## 3. Data and Research Integrity Rules (Expanded)

### 3.1 The Labeled Dataset Must Be Versioned, Never Silently Overwritten

Every meaningful change to the labeled dataset (Schema doc Section 8) produces a new versioned file (`labeled_incidents_v1.json`, `v2.json`, and so on) rather than editing the existing file in place. This is what makes every reported evaluation metric traceable back to the exact dataset it was computed against — a metric reported without a clear dataset version attached to it is not a reproducible, defensible research result.

### 3.2 Every Reported Comparative Metric Must State Its Dataset Version and Split

Any accuracy, latency, or cost figure written into the final report, presented in a demo, or included in a potential paper must be accompanied by which `datasetVersion` and which train/test split it was computed on (per Schema doc Section 4.6's `EvaluationRun.datasetVersion` field, which exists specifically to make this traceable at the data layer, not just something remembered informally).

### 3.3 Synthetic Data Generation Must Be Deterministic

Any script that programmatically generates synthetic labeled examples (Implementation Plan Phase 3) must use a fixed, recorded random seed, so that running the exact same generation script again reproduces the exact same dataset — a requirement for the dataset generation methodology to be honestly reproducible, which matters directly for the mentor's paper-worthy-research expectation.

### 3.4 All Three Engines Must Be Evaluated on the Identical Held-Out Test Examples

The train/test split created during Engine 2's development (Implementation Plan Phase 4) must be reused, unchanged, for Engine 3's evaluation (Phase 5) and for the final three-way comparison (Phase 6) — never a freshly, independently drawn sample per engine. **Why this is a hard rule, not a preference:** if each engine were evaluated on a slightly different sample of the dataset, any observed difference in accuracy between engines could be partly (or entirely) explained by the samples themselves happening to differ in difficulty, rather than by a genuine difference in the engines' reasoning quality — which would invalidate the comparison's core scientific claim.

### 3.5 Engine Failures During Evaluation Are Scored as Incorrect, Never Excluded

As specified in detail in App Flow Part B.4, if an engine fails to produce any diagnosis at all for a given labeled example during an evaluation run (a timeout, an API error, an exhausted-retries fallback), that example is recorded with `top1Correct: false` and `top3Correct: false` — it is never simply skipped or excluded from that engine's denominator. **Why:** silently excluding failed calls from an engine's accuracy calculation would let an unreliable engine's reported accuracy look artificially better than it should, since it would only ever be "graded" on the examples it successfully attempted.

---

## 4. Git and Version Control Rules (Expanded)

- Commit messages are written in plain, descriptive English explaining what changed and briefly why — never bare messages like "fix bug," "update," or "wip" with no further context.
  - **Bad:** `fix scheduler`
  - **Good:** `Fix scheduler skipping endpoints when one request in a tick throws an unhandled error — switched from Promise.all to Promise.allSettled per Tech Spec Section 4.1`
- Feature work happens on branches named `feature/<short-description>` (e.g., `feature/incident-flap-cooldown`); branches merge into `main` only once the relevant work is manually verified working, per whatever "Verify" step the Tracker document specifies for that task.
- Never commit: `.env` files, `node_modules/`, `__pycache__/`, database dump files, or trained model artifacts above a reasonable size threshold (roughly a few tens of MB).
- Large trained model files (fine-tuned LoRA adapter weights, serialized scikit-learn/XGBoost models above the size threshold) are tracked via Git LFS, or excluded from the repository entirely with a documented reproduction script (e.g., "run `train_engine2_model.py` to regenerate this file") committed in their place.

---

## 5. Testing and Validation Rules (Expanded)

### 5.1 "Done" Means Manually Verified End-to-End, Not "Code Compiles"

No task in the To-Do/Tracker document is marked `[x]` until it has been manually run through its specified "Verify" scenario and confirmed working — writing code that appears correct on inspection, without actually running the corresponding end-to-end scenario, does not satisfy this rule.

### 5.2 Diagnosis Context Schema Changes Must Be Applied Consistently, Everywhere, at Once

If the shape of the Diagnosis Context Schema (Tech Spec Section 6) ever needs to change — say, adding a new field to help engines reason about a newly-discovered failure pattern — that change must be made, in the same work session, to: `buildDiagnosisContext()`, Engine 1's prompt and parsing logic, Engine 2's feature-engineering script, Engine 3's fine-tuning data format, and the Evaluation Harness's expectations. Updating only one engine's understanding of the schema while leaving the others behind would silently break the fairness of the three-way comparison.

### 5.3 New Failure Taxonomy Categories Must Be Added Consistently, Everywhere, at Once

Similarly, if a new failure category is ever added to the canonical taxonomy (Schema doc Section 6), it must be added, in the same work session, to: the Schema doc's taxonomy table itself, Engine 1's system prompt, Engine 2's label encoding, and Engine 3's fine-tuning data — never added to just one location and left inconsistent elsewhere.

---

## 6. Scope Discipline Rules (Expanded)

### 6.1 Respect the PRD's Non-Goals Without Exception

Do not implement multi-tenant billing, enterprise-scale clustering, full incident-management workflows (on-call scheduling, paging integrations), or auto-remediation — these are explicitly out of scope per the PRD, and implementing any of them without an explicit, documented scope change agreed with the mentor represents wasted effort against the project's actual grading criteria.

### 6.2 Follow the Implementation Plan's Phase Order

Do not begin Engine 2 (Phase 4) or Engine 3 (Phase 5) work before the Phase 3 labeled dataset genuinely exists in a usable, versioned form — both engines depend on it, and building it once, properly, before either engine's work begins is strictly cheaper than improvising a partial dataset separately for each.

### 6.3 The Fallback Order Under Time Pressure

If a phase is at risk of running over its allotted time, the fallback priority order — from most to least protected — is:

1. **Phase 1 (core monitoring engine)** — protect this above everything else; it alone is a complete, gradeable, demoable project.
2. **Phase 2 (Engine 1 — prompt-based diagnosis)** — protect this second; it's what makes the "AI-powered" claim genuinely real, at relatively low implementation cost.
3. **Phase 3 (labeled dataset) and Phase 4 (Engine 2)** — protect these third; together they provide a legitimate, complete two-way AI-approach comparison even if nothing further is built.
4. **Phase 6 (evaluation harness)** — worth protecting even in a reduced two-engine form, since a comparison across two approaches, well-executed and honestly reported, is still a meaningful research contribution.
5. **Phase 5 (Engine 3 — fine-tuning)** — the single most expendable phase under time pressure; it is explicitly acceptable, per the PRD's risk table, for this to be scoped down, partially completed and reported honestly as such, or dropped entirely if time genuinely runs out.

---

## 7. Design Compliance Rule (Expanded)

All frontend work must follow `04_DESIGN.md` precisely — not just its general spirit, but its specific, enumerated constraints: the exact color palette (Section 2, with every color's permitted usage explicitly enumerated), the exact type scale (Section 3), the 4/8/16/24/32/48px spacing scale (Section 4.2), and the documented component states (Section 5). Before considering any new screen "done," it must be checked against the Design doc's Section 9 consistency checklist. If a design decision isn't explicitly covered by the Design doc, the correct default is always the simplest, calmest, least visually elaborate option available — never the most polished-looking one — consistent with the project's stated minimalist direction.

---

## 8. Documentation Upkeep Rule (Expanded)

Any architectural decision made during implementation that deviates from what's written in `02_TECH_SPEC.md` or `05_SCHEMA.md` must be reflected back into those documents in the same work session the deviation occurs — for example, if Phase 4's actual best-performing model turns out to require a change to the shared diagnosis contract's `rankedCauses` shape, that change must be written back into the Tech Spec immediately, not left as an undocumented discrepancy between "what the docs say" and "what the code actually does." The documentation set exists to be a living, accurate reflection of the real system at all times — not a static plan written once at the project's start and never revisited.

---

## 9. Priority Order When Rules Conflict

In the rare case that following one rule in this document would require violating another (for example, if strictly following the verbose-code rule in Section 1 would somehow conflict with a performance-critical piece of the scheduler in Section 2.1), the priority order is:

1. **Correctness and reliability of the core monitoring engine** (Section 2.1, 2.5, and the Phase 1 exit test in the Tracker) — this is the foundation everything else depends on, and nothing overrides it.
2. **Research integrity** (Section 3, in full) — the mentor's core ask depends on this being followed exactly, with no shortcuts.
3. **Code style and readability** (Section 1) — important, and almost never in genuine conflict with the above two, but yields to them in the rare case of true conflict (e.g., a genuinely performance-critical hot path may need a more compact form than Section 1 would otherwise prefer — but such an exception must be called out with an explicit comment explaining why, per Section 1.9's "comment the why" principle).
4. **Design compliance** (Section 7) — important for the product's usability and demo quality, but never at the expense of the three priorities above it.
