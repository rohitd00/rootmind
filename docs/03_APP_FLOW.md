# RootMind — App Flow Document (In-Depth Edition)

**Document Version:** 3.0 (Deep Expansion)
**Companion to:** 01_PRD.md, 02_TECH_SPEC.md, 05_SCHEMA.md
**Purpose:** This document is the complete behavioral specification of RootMind — every screen a user can land on, every action they can take from it, every possible outcome of that action (success, failure, and everything in between), and every background process the system runs on its own. The goal is that a developer, an AI coding agent, or a mentor reading only this document should be able to predict, with certainty, exactly what RootMind does in any situation described here — nothing should be left to guesswork or "common sense assumption" once this document has been consulted.

---

## 1. How to Read This Document

This document is organized into five parts:

1. **Part A — User-Facing Flows:** every screen-to-screen journey a logged-in user can take, written as detailed, numbered sequences, including what they see, what they can click, and what happens next in every branch.
2. **Part B — System-Level Background Flows:** everything RootMind does automatically, on a schedule or in response to data changes, without any user sitting in front of a screen waiting for it.
3. **Part C — Exhaustive Error and Edge-Case Catalogue:** a scenario-by-scenario walkthrough of what happens when something doesn't go the "happy path" way — network failures, malformed data, race conditions, and unusual user behavior.
4. **Part D — Cross-Cutting Concerns:** behaviors that apply across multiple flows at once (session handling, polling behavior, timezone handling) and are easier to describe once, centrally, than to repeat inside every individual flow.
5. **Part E — Complete Screen Inventory:** a single reference table mapping every screen to its route, its primary purpose, and which flows touch it.

---

## PART A — USER-FACING FLOWS

### A.1 First-Time Onboarding Flow (Exhaustive)

**Entry point:** An unauthenticated visitor opens the RootMind URL for the first time.

1. The visitor lands on the **Login / Register page** — a single page, not two separate marketing-style landing and login pages, with a simple toggle control at the top switching between "Log In" and "Create Account" modes. This keeps the very first thing a visitor sees functional rather than promotional, appropriate for a utility tool rather than a consumer product trying to convert visitors.
2. **If the visitor selects "Create Account":**
   a. They are shown two fields: Email and Password, plus a "Create Account" submit button.
   b. As they type, client-side validation gives immediate, inline feedback: an email-format check on the Email field (on blur, not on every keystroke, to avoid flashing an error while they're still mid-typing), and a password-strength indicator on the Password field (minimum 8 characters, at least one number — communicated as plain text beneath the field, not as a colored strength meter, per the Design doc's restrained visual language).
   c. On submit, if client-side validation passes, a request is sent to `POST /api/auth/register`.
   d. **Server-side, in order:** the email format is re-validated (never trust client-side validation alone), a check is made for whether this email is already registered, the password is hashed with bcrypt, and a new `User` row is created with `alertEmailEnabled` defaulting to `true`.
   e. **On success:** the user is redirected to the Login form (with the email field pre-filled with what they just registered, but the password field left empty) rather than being automatically logged in. This is a deliberate design choice: it means the login code path is exercised by literally every new user from day one, rather than being a rarely-tested path that only existing users take — which matters for a project graded partly on engineering correctness.
   f. **On failure (email already registered):** an inline error appears directly below the Email field: "An account with this email already exists. Try logging in instead." — with a small link that switches the form to Login mode with the email pre-filled, reducing friction for a user who forgot they'd already signed up.
   g. **On failure (server error, e.g., database temporarily unreachable):** a generic, non-technical error banner appears at the top of the form: "Something went wrong creating your account. Please try again in a moment." — the raw error is logged server-side with full detail for debugging, but never shown to the user, since exposing internal error messages is both a poor user experience and a minor security concern (it can leak implementation details).
3. **If the visitor selects "Log In" (either directly, or after registering):**
   a. They enter Email and Password and submit.
   b. A request is sent to `POST /api/auth/login`.
   c. **Server-side:** the email is looked up; if found, the submitted password is checked against the stored bcrypt hash.
   d. **On success:** a JWT is issued (containing the user's ID and a reasonable expiry, e.g., 7 days) and returned to the client.
   e. **On the client:** the JWT is stored (in-memory plus `localStorage` for persistence across page reloads, since this is a low-security-stakes student project where a more elaborate refresh-token/httpOnly-cookie scheme would be disproportionate engineering effort for the actual risk involved) and attached as an `Authorization: Bearer <token>` header on every subsequent API request.
   f. The user is redirected to `/dashboard`.
   g. **On failure (wrong password, or email not found):** a single, deliberately non-specific inline error is shown: "Incorrect email or password." — never "email not found" versus "wrong password" as two distinct messages, since revealing which one was wrong lets an attacker enumerate which emails have registered accounts, a well-known basic security practice worth following even in a student project.
4. **Immediately after first login, on `/dashboard`:** the backend query for this user's endpoints returns zero rows (a brand-new account). Rather than rendering an empty table with just column headers and no rows — which reads as a possible bug or broken page — the Dashboard explicitly detects the zero-endpoint case and renders a dedicated **empty state** (see Design doc, Section 5.7): a short centered icon, the text "You haven't added any endpoints yet," and a single prominent "Add Your First Endpoint" button.

### A.2 Add Endpoint Flow (Exhaustive)

**Entry point:** the user clicks "Add Endpoint" from either the Dashboard's empty state or its regular toolbar (once endpoints already exist).

1. A form appears — implemented as a modal overlay for a quick, low-friction feel, though a dedicated page is an equally acceptable implementation choice as long as it's used consistently everywhere in the app (per the Design doc's consistency principle).
2. **Fields presented, in order, one per row (single-column form layout per the Design doc):**
   - **Name** (required, free text) — placeholder text suggests a format: "e.g., Payments API - Production"
   - **URL** (required) — placeholder: "https://api.example.com/health"
   - **HTTP Method** (dropdown, default GET) — options: GET, POST, PUT, DELETE, PATCH, HEAD
   - **Expected Status Code** (numeric input, default 200)
   - **Check Interval** (dropdown: "Every 1 minute" / "Every 5 minutes" / "Every 15 minutes")
   - **Timeout Threshold** (numeric input in milliseconds, default 5000)
   - **Custom Headers** (optional, collapsed by default behind an "Add custom headers" expandable link, since most endpoints won't need this — showing it by default would clutter the form for the common case) — once expanded, presents key/value input pairs with an "Add another header" link
3. **Client-side validation before submit:** Name and URL must not be empty; URL must match a basic URL-format pattern (protocol + host, at minimum); Expected Status Code must be a valid HTTP status range (100–599); Timeout must be a positive number within a sane range (e.g., 1,000–30,000ms, enforced both client-side for immediate feedback and server-side as the authoritative check).
4. On submit, a request is sent to `POST /api/endpoints`. The submit button is disabled and shows a brief loading state ("Testing connection...") — the user should never be able to double-submit by clicking twice while a request is in flight.
5. **Server-side processing, in order:**
   a. Full validation is re-run (never trust the client alone).
   b. A **one-time synchronous test ping** is issued to the provided URL using the given method, headers, and timeout — this happens before anything is written to the database.
   c. **If the test ping succeeds** (a response arrives within the timeout and its status code matches `expectedStatus`): the endpoint is created with `isActive = true` and `nextCheckAt` set to the current time (so it gets picked up on the scheduler's very next tick, rather than waiting a full interval before the first real check). The API responds with success and the new endpoint's data.
   d. **If the test ping fails for any reason** (wrong status code, timeout, DNS failure, connection refused, SSL error): the API does **not** save the endpoint automatically. Instead, it responds with a structured "test failed" response describing exactly what happened (e.g., `{ "testResult": "failed", "reason": "TIMEOUT", "detail": "No response received within 5000ms" }`).
6. **On the frontend, when the test ping failed:** rather than a generic error banner, the form itself displays the specific reason inline near the URL field (e.g., "Received status 404, expected 200" or "Request timed out after 5000ms — is the URL correct and publicly reachable?"), and presents **two explicit buttons**: **"Fix and Retry"** (keeps the form open with all entered values intact, so the user doesn't have to retype anything, and focus moves to the URL field for quick correction) and **"Save Anyway"** (submits again with an explicit `forceSave: true` flag, telling the backend to save the endpoint despite the failed test — appropriate for endpoints that are legitimately behind auth the user hasn't finished configuring yet, or that are known to be temporarily down).
7. **On successful save (either path):** the modal closes, and the Dashboard refreshes to show the new endpoint. If it was saved via the normal successful-test path, its status badge shows "Pending first scheduled check" until the scheduler's next tick actually runs a check against it (which, given `nextCheckAt` was set to "now," happens within roughly 30 seconds — the scheduler's tick interval). If it was saved via "Save Anyway" after a failed test, the badge still shows "Pending first scheduled check" — the failed *test* ping is not itself recorded as the endpoint's first official `Check` row; the real first check happens on the next scheduler tick, keeping the check history clean and consistent regardless of how the endpoint was added.

### A.3 Viewing Overall Health — Dashboard Flow (Exhaustive)

**Entry point:** the user navigates to `/dashboard`, either directly after login or via the sidebar.

1. On page load, a request is sent to `GET /api/endpoints`, returning every active endpoint belonging to the logged-in user, along with each one's current status (computed server-side from its most recent `Check` row and any open `Incident`), 24-hour uptime percentage, and average response time over the same window.
2. **While this request is in flight:** the page shows a lightweight loading skeleton (simple gray placeholder blocks in the shape of the eventual table rows) rather than a blank white screen or a spinning loader icon — skeleton loading communicates "content is coming, and roughly how much" more informatively than a generic spinner.
3. **Once data arrives:**
   - **If zero endpoints exist:** the empty state described in Flow A.1, step 4, is shown.
   - **If one or more endpoints exist:** they are rendered as a list of rows, sorted with unhealthy endpoints (any currently Down, then any currently Degraded) at the top, followed by healthy (Up) endpoints below, so a user scanning the page during an active problem sees exactly what needs attention first without scrolling or searching.
4. Each row displays: the endpoint's name, its status badge, its 24-hour uptime percentage (e.g., "99.2%"), and its average response time over the same window (e.g., "184ms").
5. **Automatic refresh:** the Dashboard polls `GET /api/endpoints` again every 30 seconds for as long as the page remains open and in the foreground (polling is paused when the browser tab is backgrounded, using the Page Visibility API, to avoid unnecessary load and battery/network usage when the user isn't actually looking at the page — resuming immediately when the tab regains focus).
6. **Clicking any row** navigates to that endpoint's Endpoint Detail page (`/endpoints/:id`).
7. **If the `GET /api/endpoints` request itself fails** (e.g., a network error, or the backend is temporarily unreachable): the page shows a calm inline error message ("Couldn't load your endpoints — check your connection and try again") with a manual "Retry" button, rather than crashing or showing a blank page. The automatic 30-second polling continues to retry in the background regardless, so if connectivity is restored, the page will recover on its own even if the user doesn't click Retry.

### A.4 Endpoint Detail Flow (Exhaustive)

**Entry point:** the user clicks an endpoint row from the Dashboard, or navigates directly to `/endpoints/:id`.

1. A request is sent to `GET /api/endpoints/:id`, returning the endpoint's full configuration, current status, and summary statistics.
2. In parallel, a request is sent to `GET /api/endpoints/:id/checks` (paginated, most recent first) to populate the recent-checks table and the response-time graph's default (24-hour) view.
3. **Page layout, top to bottom:**
   a. **Header:** endpoint name and URL, status badge, and a "since" duration ("Up for 3 days, 4 hours" or "Down since 14:02 today").
   b. **Response-time graph:** defaults to the last 24 hours; a toggle switches to the last 7 days, which triggers a new `GET /api/endpoints/:id/checks?range=7d` request. Failed checks are marked as small red points along the same timeline (per the Design doc).
   c. **Recent checks table:** paginated, 20 rows per page by default, showing timestamp, status code, response time, and success/fail for each. A "Load more" control (not full pagination controls with page numbers, since users are far more likely to want "a bit more history" than to jump to a specific page number) appends the next 20 rows.
   d. **Incident history list:** every past incident for this specific endpoint, most recent first, each summarized with its duration and — once diagnosis has completed — a one-line probable-cause summary.
4. **Available actions from this page:**
   - **"Edit Configuration"** opens the same form used in Flow A.2, pre-filled with the endpoint's current values. Saving changes updates the endpoint via `PUT /api/endpoints/:id`. Note: editing configuration does **not** re-run the one-time test ping from Flow A.2 — the assumption is that an already-monitored endpoint doesn't need to be re-validated the same way a brand-new one does, since its ongoing check history already demonstrates whether it's reachable.
   - **"Pause Monitoring"** sets `isActive = false` via `PUT /api/endpoints/:id`. A confirmation dialog explains: "This endpoint will stop being checked until you resume it. Its history will be preserved." The endpoint's row on the Dashboard (once resumed) or a dedicated "Paused Endpoints" filter (a reasonable stretch enhancement, not required for MVP) reflects this state.
   - **"Delete Endpoint"** also sets `isActive = false` (a soft delete, per the Schema doc's rationale) but is presented to the user as a distinct, more final-sounding action with a red destructive-style confirmation dialog: "This will remove [endpoint name] from your dashboard. Its historical data will be retained but it will no longer be checked." This wording is deliberately honest about what actually happens (soft delete, not permanent erasure) rather than implying data is destroyed, since the underlying behavior genuinely does preserve history.

### A.5 Incident Detection Becoming Visible to the User (Exhaustive)

This flow describes the moment where a system-level process (detailed in Part B) first becomes visible on screen.

1. **Scenario: the user has the Dashboard open in a browser tab** when, in the background, an endpoint they monitor crosses the consecutive-failure threshold and an incident opens.
2. On the Dashboard's next 30-second polling refresh, the `GET /api/endpoints` response now reflects the new status for that endpoint. The row re-renders: its badge changes from green "Up" to red "Down" (or amber "Degraded," if the failure was a latency-threshold breach rather than a hard failure), and — per Flow A.3, step 3 — the row automatically moves toward the top of the sorted list on the next render, since unhealthy endpoints are always sorted first.
3. **No jarring visual effect accompanies this change** (no flashing, no sound, no browser notification popup) — consistent with the Design doc's "calm, not alarming" principle; the color change itself, combined with the row reordering, is sufficient signal.
4. **Scenario: the user does not have the Dashboard open at all.** If email alerting is enabled (`alertEmailEnabled = true` on their account), they instead learn about the incident via the alert email sent by the background Diagnosis Orchestrator flow (Part B.3), independent of whether they're looking at the app.
5. **From either path**, the user's natural next action is to click into the affected endpoint (from the Dashboard) or directly into the Incidents list (from the sidebar) to investigate further, leading into Flow A.6 or A.4.

### A.6 Incidents List Flow (Exhaustive)

**Entry point:** the user clicks "Incidents" in the sidebar.

1. A request is sent to `GET /api/incidents?status=open` (the default filter, since open incidents are almost always what a user visiting this page cares about most urgently).
2. **Filter controls** at the top of the page let the user switch between "Open" and "Resolved," and optionally narrow by a date range (useful for reviewing history, e.g., "show me everything from last week" during a retrospective).
3. **Each incident is rendered as a card** (per the Design doc's Incident Card component): endpoint name as the title, a colored left border (red for open, gray for resolved), a subtitle showing duration/timing, and — if a diagnosis has completed — a one-line summary of the top probable cause; if not yet complete, "Diagnosis pending..." in plain secondary-gray text.
4. **If the filtered list is empty** (e.g., the user filters to "Open" and genuinely has no open incidents right now): a small, calm empty state is shown — "No open incidents. Everything's healthy." — which is itself a meaningful, reassuring piece of information for a monitoring tool, not just a fallback for "nothing to show."
5. **Clicking any incident card** navigates to that incident's Incident Detail page.

### A.7 Incident Detail Flow, Including the Diagnosis Panel (Exhaustive)

**Entry point:** the user clicks an incident card from the Incidents list, or from an endpoint's incident history, or via a direct link (e.g., from an alert email).

1. A request is sent to `GET /api/incidents/:id`, returning the incident's timeline data, the specific `Check` rows that triggered and occurred during it, and any associated `Diagnosis` rows.
2. **Page layout, top to bottom:**
   a. **Timeline header:** "Opened at [timestamp]" and, if resolved, "Resolved at [timestamp] — lasted [duration]"; if still open, "Ongoing — [elapsed duration] so far."
   b. **Raw evidence table:** the specific checks involved, shown exactly as recorded (timestamp, status code, response time, error type) — included specifically so a technically-minded user, or a mentor during a demo, can independently verify that the diagnosis panel's conclusions are actually consistent with the real underlying data, rather than having to take the AI's explanation on faith.
   c. **Diagnosis Panel(s):** if more than one `Diagnosis` row exists for this incident (because the user has manually re-run diagnosis with multiple engines), each is shown as its own distinct panel, stacked vertically, each clearly labeled with its engine tag — enabling direct visual side-by-side comparison, which is exactly the capability the PRD's research objective calls for during live demos.
3. **Within each Diagnosis Panel:**
   - If `status = "completed"`: the ranked causes (confidence bars), the full explanation paragraph, and the suggested next step are all shown per the Design doc's layout.
   - If `status = "pending"` (the async orchestration hasn't finished yet): a lightweight "Generating diagnosis..." placeholder is shown; the page's own polling (every 5–10 seconds, more frequent than the Dashboard's 30-second interval, since a user actively viewing an Incident Detail page is more likely waiting specifically for this to resolve) will replace it automatically once the diagnosis completes, without requiring a manual page refresh.
   - If `status = "unavailable"` (the engine failed or timed out after retries): a plain, honest message is shown — "Diagnosis unavailable for this attempt." — with a "Retry" button that re-triggers `POST /api/incidents/:id/diagnose` for the same engine.
4. **"Re-run diagnosis" control:** a dropdown (Prompt-based LLM / Custom ML Model / Fine-tuned LLM — only listing engines that have actually been implemented so far, per the project's phased build order) plus a "Run" button. Clicking "Run" sends `POST /api/incidents/:id/diagnose?engine=<selected>`, and the new resulting Diagnosis Panel appears below any existing ones once complete — existing diagnoses are never overwritten or removed, since preserving every attempt is valuable both for comparison and for the Evaluation Harness's broader research purpose (even though this specific manual-comparison data isn't itself part of the formal evaluation dataset).
5. **If the incident is still open** when the user is viewing this page and it resolves while they're looking at it (detected via the page's own polling), the timeline header updates in place from "Ongoing" to "Resolved at [time] — lasted [duration]" without requiring a manual reload.

### A.8 Incident Resolution Flow — User-Visible Side (Exhaustive)

1. Independent of any user action, once the affected endpoint passes a subsequent scheduled check (see Part B.2 for the exact background logic, including flap-cooldown handling), the system marks the incident `resolved`.
2. **If the user has the relevant Incident Detail page open:** per Flow A.7, step 5, the page updates in place on its next poll.
3. **If the user has the Dashboard open:** the endpoint's badge returns to green "Up" on the Dashboard's next 30-second poll, and the row's sort position moves back down among the healthy endpoints.
4. **If the user has the Incidents list open, filtered to "Open":** the resolved incident disappears from that filtered view on the next poll (since it no longer matches `status=open`) — appearing instead if the user switches the filter to "Resolved."
5. **If email alerting is enabled:** a resolution email is sent, using the same visual/textual style as the original alert email but with resolved-state framing ("Good news — [endpoint name] has recovered. It was down for [duration]."), closing the loop for a user who may not be actively watching the dashboard at all.

### A.9 Settings Flow (Exhaustive)

**Entry point:** the user clicks "Settings" in the sidebar.

1. A request is sent to `GET /api/settings` (or equivalently, settings are included in a `/api/auth/me`-style endpoint) returning the user's email and current `alertEmailEnabled` value.
2. The page shows the email as read-only text (changing email addresses is not a required feature for this project's scope — if genuinely needed later, it would require its own dedicated, more careful flow involving re-verification, which is out of scope here) and a single toggle switch labeled "Email alerts for incidents."
3. **Toggling the switch** immediately sends `PUT /api/settings` with the new value — there is no separate "Save" button for this single, low-risk setting, since requiring an extra confirmation step for a single boolean toggle would be unnecessary friction.
4. **On success:** the toggle's new state is confirmed visually (it simply reflects the new position — no additional toast/confirmation message is needed for such a low-stakes action).
5. **On failure** (e.g., a network error mid-toggle): the toggle visually reverts to its previous state, and a small inline message appears: "Couldn't save that change — please try again."

### A.10 Evaluation / Comparative Results Flow (Exhaustive)

**Entry point:** the user (in practice, the project author, or the author walking the mentor through a demo) navigates to `/evaluation`.

1. **If no evaluation run has ever been completed:** an empty state is shown — "No evaluation results yet. Run a comparison to see how the three diagnosis engines perform." — with a "Run Comparative Evaluation" button.
2. **If at least one prior run exists:** the results of the most recent completed run are shown by default, with a dropdown allowing the user to select and view results from any earlier run instead (supporting the case where the labeled dataset has been improved over time and the author wants to show progression across dataset versions — see Schema doc, Section 4.6, on `datasetVersion`).
3. **Clicking "Run Comparative Evaluation":**
   a. A confirmation dialog appears: "This will run all three engines against the full labeled dataset ([N] examples) and may take several minutes due to LLM API latency. Continue?"
   b. On confirmation, `POST /api/evaluation/run` is called, and the button is replaced with a progress indicator.
   c. **The frontend polls a progress-check endpoint** (or the same `GET /api/evaluation/results` endpoint, checking whether `completedAt` is still null) every few seconds, updating a simple textual progress readout ("Evaluating... 142 of 300 examples processed") — the exact number processed can be derived from counting `EvaluationResult` rows already written for the current, in-progress `EvaluationRun`.
4. **On completion**, the page renders:
   - A **summary table**: one row per engine, columns for top-1 accuracy, top-3 accuracy, average latency, and estimated cost per 1,000 diagnoses.
   - A **latency comparison bar chart** and a **cost comparison bar chart**, both using the single accent-blue color per the Design doc (these are neutral informational charts, not status charts, so they don't use the red/amber/green status palette).
   - **Expandable confusion matrices**, one per engine, collapsed by default (since they're the most detailed, least-often-needed view) with a "View confusion matrix" toggle per engine.
5. **"Export Results" button:** triggers a client-side download of the full result set as both a `.csv` and a `.json` file (offered as two separate download links, since a CSV is more convenient for quick spreadsheet inspection while JSON preserves the full nested structure — e.g., full ranked-cause lists — more faithfully for anyone processing it programmatically for the paper).
6. **If an evaluation run is triggered but fails partway through** (e.g., the LLM API becomes unreachable mid-run): the `EvaluationRun` row is left with `completedAt = null` and whatever `EvaluationResult` rows were successfully written remain in place; the frontend shows: "This evaluation run didn't complete — [X] of [N] examples were processed before an error occurred. You can view partial results below or start a new run." — partial results are still genuinely useful (e.g., if the failure happened late in the run) rather than being discarded outright.

---

## PART B — SYSTEM-LEVEL (BACKGROUND) FLOWS

### B.1 Health Check Scheduler Loop (Exhaustive)

```
Every 30 seconds, on a node-cron tick:

  STEP 1 — Query for due endpoints:
    SELECT * FROM endpoints
    WHERE is_active = true AND next_check_at <= NOW()

  STEP 2 — For each due endpoint, independently and in parallel
           (using Promise.allSettled, NOT Promise.all, specifically so that
           one endpoint's request failure/exception can never prevent the
           others in the same tick from completing):

    a. Build the HTTP request from the endpoint's stored configuration:
       method, url, headers (parsed from the customHeaders JSON field),
       and timeout.

    b. Send the request via axios, wrapped in try/catch.

    c. IF a response is received:
         IF response.statusCode == endpoint.expectedStatus:
             Record Check { success: true, statusCode, responseTimeMs }
         ELSE:
             Record Check { success: false, statusCode, responseTimeMs,
                             errorType: "NON_2XX",
                             responseSnippet: first 500 chars of body,
                             responseHeaders: response headers }

    d. IF the request throws before any response is received:
         Classify the thrown error into one of:
           - "TIMEOUT"              (axios timeout exceeded)
           - "CONNECTION_REFUSED"   (ECONNREFUSED)
           - "DNS_FAILURE"          (ENOTFOUND / EAI_AGAIN)
           - "SSL_ERROR"            (certificate/handshake failure)
         Record Check { success: false, errorType: <classified type>,
                         responseTimeMs: null or measured time until failure,
                         statusCode: null }

    e. Update endpoint.nextCheckAt = NOW() + endpoint.checkIntervalSec

    f. IF the check failed (success: false):
         Call IncidentManager.handleFailedCheck(endpointId, checkId)
       ELSE (success: true):
         Call IncidentManager.handleSuccessfulCheck(endpointId, checkId)

  STEP 3 — After all endpoints in this tick have been processed
           (successfully or not), log a single heartbeat line:
    "[timestamp] Tick complete — N endpoints checked, M failures, K skipped-due-to-error"
```

**Why `Promise.allSettled` specifically, spelled out:** if the scheduler used `Promise.all` instead, a single endpoint whose request handling threw an *unexpected* (not properly caught) error could cause the entire batch's promise chain to reject, potentially skipping the `nextCheckAt` update and heartbeat logging for every other endpoint in that tick — a serious reliability bug for a system whose entire value proposition rests on checks actually running on schedule. `Promise.allSettled` guarantees every endpoint's processing runs to completion (or fails) independently.

### B.2 Incident Manager Logic (Exhaustive)

```
FUNCTION handleFailedCheck(endpointId, checkId):
  consecutiveFailures = count of most recent consecutive Check rows
                         for this endpoint where success = false
                         (counting backward from the most recent check
                          until a success is hit or history is exhausted)

  IF consecutiveFailures < FAILURE_THRESHOLD (default 2):
      // Not yet confirmed — could still be a transient blip.
      Do nothing further. The failed Check row itself is already
      recorded; that's sufficient for now.
      RETURN

  IF consecutiveFailures >= FAILURE_THRESHOLD:
      existingOpenIncident = find Incident WHERE endpointId = ? AND status = "open"

      IF existingOpenIncident EXISTS:
          // Already tracked — this failure is just part of the ongoing incident.
          Do nothing further (no duplicate incident is created).
          RETURN

      ELSE:
          newIncident = CREATE Incident {
              endpointId: endpointId,
              status: "open",
              openedAt: NOW()
          }
          // Fire-and-forget: does not block the scheduler's own execution.
          Async call: DiagnosisOrchestrator.diagnoseIncident(newIncident.id)
          RETURN


FUNCTION handleSuccessfulCheck(endpointId, checkId):
  openIncident = find Incident WHERE endpointId = ? AND status = "open"

  IF openIncident DOES NOT EXIST:
      // Nothing to resolve — this endpoint wasn't in a failing state.
      RETURN

  IF openIncident EXISTS:
      recentFlapWindowSeconds = FLAP_COOLDOWN_SECONDS (default 120)
      hasFailedAgainRecently = check whether this endpoint has ANY
                                failed Check row within the last
                                recentFlapWindowSeconds, EXCLUDING
                                the just-recorded successful check

      IF hasFailedAgainRecently:
          // Treat this success as part of an ongoing flap rather than
          // a genuine, stable recovery — wait for a more sustained
          // success before closing the incident.
          Do nothing further this tick.
          RETURN

      ELSE:
          UPDATE openIncident SET
              status = "resolved",
              resolvedAt = NOW(),
              durationSec = (NOW() - openIncident.openedAt) in seconds

          IF user.alertEmailEnabled:
              Async call: AlertService.sendResolutionEmail(openIncident.id)
          RETURN
```

**Worked example of the flap-cooldown logic:** suppose an endpoint fails at 14:00:00 and 14:01:00 (opening an incident at 14:01:00, since that's the 2nd consecutive failure), then succeeds at 14:02:00. Because a failure occurred at 14:01:00 — well within the 120-second flap-cooldown window counting backward from the 14:02:00 success — `hasFailedAgainRecently` evaluates true, and the incident is **not** yet resolved on this success alone. If the endpoint then succeeds again at 14:03:00, and no failure has occurred in the 120 seconds prior to that (i.e., nothing since the 14:01:00 failure), the incident resolves at 14:03:00. This prevents an endpoint that's genuinely flapping (failing and recovering every minute or two) from generating a confusing string of open-then-immediately-resolved incidents.

### B.3 Diagnosis Orchestration Flow (Exhaustive)

```
FUNCTION diagnoseIncident(incidentId, engineOverride = null):

  STEP 1 — Build the context payload:
    contextPayload = buildDiagnosisContext(incidentId)
    // See Tech Spec Section 6 for the exact schema this produces.
    // This is the ONLY function in the entire codebase permitted to
    // construct this payload, guaranteeing every engine call — whether
    // in production, manual re-run, or evaluation — receives an
    // identically-shaped payload built the identical way.

  STEP 2 — Determine which engine(s) to call:
    IF engineOverride is provided (manual re-run from the UI):
        enginesToCall = [engineOverride]
    ELSE (automatic, on new incident):
        enginesToCall = [DEFAULT_PRODUCTION_ENGINE]  // "prompt", per current config

  STEP 3 — For each engine in enginesToCall:
      TRY:
          startTime = NOW()
          result = engine.diagnose(contextPayload)  // includes engine-specific
                                                       // retry logic internally,
                                                       // e.g. Engine 1's up-to-2
                                                       // retries on malformed JSON
          elapsedMs = NOW() - startTime

          CREATE Diagnosis {
              incidentId: incidentId,
              engineUsed: engine.name,
              status: "completed",
              rankedCauses: result.rankedCauses,
              explanation: result.explanation,
              suggestedNextStep: result.suggestedNextStep,
              latencyMs: elapsedMs
          }

      CATCH (any error, including exhausted retries or timeout):
          CREATE Diagnosis {
              incidentId: incidentId,
              engineUsed: engine.name,
              status: "unavailable",
              rankedCauses: [],
              explanation: null,
              suggestedNextStep: null,
              latencyMs: null
          }
          LOG the underlying error with full detail (incidentId, engine name,
              error message/stack) for later debugging — this is never
              shown to the end user, but is essential for the developer
              to diagnose why an engine failed during development or demo prep

  STEP 4 — IF this was the automatic (non-override) path AND user.alertEmailEnabled:
      Async call: AlertService.sendIncidentOpenEmail(incidentId)
      // Note: this email is sent regardless of whether diagnosis succeeded —
      // if it completed in time, the email includes a short diagnosis summary;
      // if not, the email is sent without one rather than being delayed
      // indefinitely waiting for a diagnosis that might never complete.
```

### B.4 Evaluation Harness Run (Exhaustive)

```
FUNCTION runEvaluationHarness(datasetFilePath):

  STEP 1:
    dataset = load and parse the labeled dataset JSON file at datasetFilePath
    evaluationRun = CREATE EvaluationRun {
        startedAt: NOW(),
        datasetSize: dataset.length,
        datasetVersion: extracted from the filename or an internal field, e.g. "v1"
    }

  STEP 2 — For each labeledExample in dataset:
      FOR each engine in [promptEngine, mlEngine, finetunedEngine]:
          TRY:
              startTime = NOW()
              result = engine.diagnose(labeledExample.contextPayload)
              elapsedMs = NOW() - startTime

              topPredictedCause = result.rankedCauses[0].cause
              top3PredictedCauses = result.rankedCauses.slice(0, 3).map(c => c.cause)

              top1Correct = (topPredictedCause == labeledExample.groundTruthCause)
              top3Correct = top3PredictedCauses.includes(labeledExample.groundTruthCause)

              CREATE EvaluationResult {
                  evaluationRunId: evaluationRun.id,
                  engineUsed: engine.name,
                  labeledExampleId: labeledExample.labeledExampleId,
                  groundTruthCause: labeledExample.groundTruthCause,
                  predictedCauses: result.rankedCauses,
                  top1Correct: top1Correct,
                  top3Correct: top3Correct,
                  latencyMs: elapsedMs
              }

          CATCH (engine failure on this specific example):
              CREATE EvaluationResult {
                  evaluationRunId: evaluationRun.id,
                  engineUsed: engine.name,
                  labeledExampleId: labeledExample.labeledExampleId,
                  groundTruthCause: labeledExample.groundTruthCause,
                  predictedCauses: [],
                  top1Correct: false,
                  top3Correct: false,
                  latencyMs: -1  // sentinel value indicating a failed call,
                                  // excluded from average-latency calculations
                                  // but still counted against accuracy —
                                  // a failure to produce any diagnosis is
                                  // correctly scored as incorrect, not
                                  // silently skipped, since silently skipping
                                  // failed calls would artificially inflate
                                  // an engine's reported accuracy
              }
              CONTINUE to the next engine/example — one failure never halts
              the overall run

  STEP 3 — After all examples and engines are processed:
      UPDATE evaluationRun SET completedAt = NOW()
      RETURN aggregated summary (computed on read, not stored redundantly):
          for each engine: top1 accuracy %, top3 accuracy %,
          average latency (excluding sentinel -1 values),
          confusion matrix (groundTruthCause x predictedCause[0], counts)
```

**Why failed evaluation calls are scored as incorrect rather than excluded:** if an engine's own unreliability (e.g., frequent API timeouts) simply removed those examples from its accuracy calculation entirely, an engine that failed to respond at all on its hardest examples could end up with an artificially inflated accuracy score — technically "correct" on every example it actually answered, while quietly never attempting the ones it would likely have gotten wrong anyway. Counting failures as incorrect keeps the comparison honest and directly comparable across all three engines, which matters given the mentor's explicit research-integrity expectations.

---

## PART C — EXHAUSTIVE ERROR AND EDGE-CASE CATALOGUE

| # | Scenario | Detailed Behavior |
|---|---|---|
| C.1 | User tries to add an endpoint whose URL is already monitored (possibly with different configuration) | Allowed without restriction — a user may legitimately want to monitor the same URL under two different configurations (e.g., one check every 1 minute expecting a 200, another every 15 minutes expecting a specific response header) — no uniqueness constraint exists on `url` |
| C.2 | An endpoint becomes permanently unreachable (e.g., the service behind it is decommissioned) | RootMind does not auto-pause or auto-delete it — checks continue indefinitely on schedule, and incidents will keep opening (and either staying open indefinitely, or opening/closing if the DNS itself intermittently still resolves to something). Deciding an endpoint is "permanently" gone and should be paused/removed is left entirely to the user's judgment, since RootMind has no reliable way to distinguish "permanently gone" from "very long, unusual outage" |
| C.3 | The active diagnosis engine (e.g., Groq API) is down, rate-limited, or returns malformed output beyond retry limits | The Diagnosis Orchestrator's per-engine try/catch (Part B.3) catches this, and a `Diagnosis` row with `status = "unavailable"` is persisted. Critically, this never blocks or corrupts the underlying `Incident` record itself — a user can always see that an incident happened and when, even if no AI explanation was ever successfully produced for it |
| C.4 | An endpoint flaps rapidly (fails, recovers, fails, recovers, all within a couple of minutes) | The flap-cooldown logic (Part B.2, worked example included) merges these into a single ongoing incident rather than creating a new incident every time it flips state, keeping the Incidents list meaningful rather than flooded with near-duplicate entries |
| C.5 | User deletes (soft-deletes) an endpoint that has open incidents | The endpoint's `isActive` becomes `false`; the scheduler simply stops selecting it in its next-due query (Part B.1, Step 1). Its existing `Incident` and `Diagnosis` rows are entirely untouched and remain fully viewable via the Incidents list and via direct links, since no cascade-delete is ever triggered by an endpoint soft-delete |
| C.6 | The Evaluation Harness is triggered while the production system is simultaneously generating live diagnoses for a real incident | No interference occurs whatsoever — the harness (Part B.4) reads only from the static dataset file and writes only to `EvaluationRun`/`EvaluationResult`, which share no foreign keys or code paths with the live `Incident`/`Diagnosis` tables and the Diagnosis Orchestrator (Part B.3) used for real incidents |
| C.7 | A user's JWT expires while they have the Dashboard open and mid-session | The next API call (whether a manual action or the routine 30-second poll) receives a 401 Unauthorized response. A global response interceptor on the frontend catches any 401, clears the locally stored JWT, and redirects to the Login page with a message: "Your session expired — please log in again." No confusing blank page or silently-failing polling loop is left running |
| C.8 | A "Save Anyway" endpoint (added despite a failed test ping, per Flow A.2) genuinely never comes online at all | Treated identically to any other endpoint from that point forward — its first real scheduled check will simply fail, and if it fails twice consecutively, a genuine incident opens and diagnosis runs against it exactly as it would for any endpoint that was working fine and later broke |
| C.9 | Two browser tabs are open simultaneously, both logged in as the same user, and the user pauses an endpoint in one tab | The other tab does not update instantly (no real-time push/WebSocket mechanism exists in this design), but will reflect the change on its own next 30-second poll — an acceptable, deliberate trade-off given the added complexity a real-time sync layer would introduce for marginal benefit at this project's scale |
| C.10 | The labeled dataset file referenced by an Evaluation Harness run is missing, malformed, or empty | `POST /api/evaluation/run` fails fast with a clear error before creating any `EvaluationRun` row at all — "Labeled dataset file not found or invalid: [error detail]" — surfaced to the user/developer rather than silently starting a run against zero or corrupted examples and producing a meaningless empty or garbage result set |
| C.11 | An endpoint's `customHeaders` includes a genuinely malformed value (e.g., not valid JSON, somehow bypassing client-side validation) | Server-side validation on `POST`/`PUT /api/endpoints` rejects the request with a 400 and a specific validation error before it ever reaches the database — the scheduler is never allowed to encounter malformed header data mid-tick, since that would risk an unhandled exception during a scheduled check |
| C.12 | A user attempts to register with an email in an unusual but technically valid format (e.g., containing a `+` alias, like `user+test@example.com`) | Accepted without special-casing — the email validation regex follows standard, permissive email-format rules rather than an overly strict pattern that might reject legitimate addresses |
| C.13 | The Groq API (or whichever LLM provider is active) changes its response format unexpectedly (a provider-side breaking change) | Engine 1's JSON schema validation step (Tech Spec, Section 4.4) catches the malformed/unexpected shape, triggers its built-in retry logic (up to 2 retries), and ultimately falls back to `status = "unavailable"` if retries are exhausted — the system degrades gracefully rather than crashing or producing a corrupted `Diagnosis` row |
| C.14 | A user tries to manually re-run diagnosis (Flow A.7) using an engine that hasn't been implemented yet at the current phase of the project (e.g., Engine 3 during Phase 4, before Phase 5 is complete) | The engine-selection dropdown on the Incident Detail page only ever lists engines that are actually implemented and wired up at the current point in development — this is a frontend/config concern (the dropdown's available options are driven by a small config list, not hardcoded to always show all three), not something the backend needs special-case error handling for, since the option to select an unbuilt engine simply never appears in the UI to begin with |
| C.15 | Two scheduler ticks somehow overlap (e.g., one tick's processing takes longer than 30 seconds due to unusually slow endpoints) | Each endpoint's `nextCheckAt` is only updated after its own check completes, and the query in Step 1 of Part B.1 naturally excludes any endpoint whose `nextCheckAt` hasn't yet passed — so even if ticks overlap in wall-clock time, no endpoint is ever checked twice in immediate succession for the same due period; at worst, a very slow endpoint's next check is checked slightly late, which is a benign, self-correcting delay rather than a data-integrity problem |

---

## PART D — CROSS-CUTTING CONCERNS

### D.1 Polling Intervals, Summarized

| Screen | Poll Interval | Paused When |
|---|---|---|
| Dashboard | 30 seconds | Browser tab is backgrounded (Page Visibility API) |
| Incidents List | 30 seconds | Browser tab is backgrounded |
| Incident Detail (while diagnosis is pending) | 5–10 seconds | Diagnosis has completed or failed for all in-flight engines; browser tab is backgrounded |
| Evaluation Results (while a run is in progress) | 3–5 seconds | Run has completed; browser tab is backgrounded |

### D.2 Timezone Handling

All timestamps are stored in UTC in the database (Postgres `DateTime` fields, per the Schema doc). The frontend converts every displayed timestamp to the user's local browser timezone at render time — the user should never need to mentally convert UTC times themselves. Relative/human-friendly phrasing ("3 minutes ago," "Down since 14:02") is preferred over raw timestamps wherever it improves quick scanning, with the exact timestamp available on hover as a tooltip for anyone who wants precision.

### D.3 Loading and Empty States, as a General Pattern

Every screen in RootMind that fetches data follows the same three-state pattern, consistently: a **loading state** (skeleton placeholders, never a blank white screen), a **populated state** (the real content), and an **empty state** (a specific, calm, actionable message — never just "no data" with nothing else, and never a table with zero rows and only column headers, which reads as a possible bug rather than a deliberate, expected state).

### D.4 Confirmation Dialogs — When They're Required

A confirmation dialog is required before: deleting an endpoint, pausing an endpoint (a lighter-weight confirmation than delete, but still present since it stops active monitoring), and triggering a full Evaluation Harness run (since it may take several minutes and consume LLM API quota). A confirmation dialog is deliberately **not** required for: toggling the email-alert setting, editing endpoint configuration (saving simply takes effect), or re-running diagnosis on an incident (a low-cost, easily-repeatable, non-destructive action).
