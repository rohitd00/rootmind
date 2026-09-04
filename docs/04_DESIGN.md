# RootMind — Design Document (In-Depth Edition)

**Document Version:** 3.0 (Deep Expansion)
**Companion to:** 01_PRD.md, 03_APP_FLOW.md
**Design Philosophy:** RootMind is a monitoring tool, not a consumer product competing for attention. Its job is to be trustworthy, calm, and fast to read during exactly the moment a user is most stressed — an active outage. Every decision in this document exists to serve one idea: **nothing on screen should compete with the data for the user's attention.** This edition goes further than a typical design brief — it walks through the reasoning for every choice, specifies exact values wherever a value can be specified, and works through concrete scenarios (a healthy dashboard, a dashboard mid-outage, a slow LLM response, a very long list of ranked causes) so that an implementer never has to guess how a real, messy situation should look.

---

## 1. Design Principles, Expanded With Concrete Consequences

### 1.1 Clarity Over Decoration

**The principle:** every visual element must communicate status or data. If it doesn't, remove it.

**What this looks like in practice:** no decorative icons next to menu items purely for visual balance (icons are used only where they add genuine, faster recognition — e.g., a small "+" on the Add Endpoint button, or a status dot on a badge). No background patterns, no card hover "lift" shadows that serve no informational purpose. When in doubt during implementation, the test to apply is: *if I removed this element, would the user understand the screen slightly less well?* If the honest answer is no, the element doesn't belong.

### 1.2 Calm, Restrained Color Usage

**The principle:** color is reserved almost entirely for status signaling.

**Worked scenario — a dashboard with 12 endpoints, 2 of which are down:** the two Down rows show a red status badge; the ten healthy rows show a green status badge. Nothing else on the page uses red or green — not the sidebar, not the "Add Endpoint" button, not the graph lines. A user's eye is drawn to exactly the two rows that need attention, because red appears nowhere else competing for that same visual weight.

### 1.3 Generous Whitespace, Especially Around Status

**Worked scenario — the Incident Detail page during an active outage:** the Diagnosis Panel's ranked-causes section has at least 24px of vertical spacing between it and the raw-evidence table above it, and at least 24px before the explanation paragraph below it. This isn't arbitrary — it ensures that in a moment where the user may be anxious and reading quickly, each distinct piece of information (the raw facts, then the AI's ranked guesses, then its reasoning) is visually separated enough that they can't accidentally blur together while skimming.

### 1.4 Consistent Hierarchy Across Every Screen

**The test:** if a user were shown a cropped screenshot of any card, table, or button from any screen in the app, they should be able to tell it's part of RootMind without needing to see the surrounding navigation — because the type scale, spacing, and card treatment are identical everywhere.

### 1.5 Legible at a Glance, Rewarding on Inspection

**Two-second test (Dashboard):** a user glancing at the Dashboard for two seconds should be able to answer "is anything wrong right now?" — this is why unhealthy endpoints sort to the top (App Flow, Flow A.3) and use a color that stands out against the otherwise neutral palette.

**Thirty-second test (Incident Detail):** a user spending thirty seconds reading an Incident Detail page should come away understanding not just *that* something broke, but a plausible *why* and *what to check next* — this is why the explanation and suggested-next-step text are never truncated or hidden.

### 1.6 Minimalism as a Constraint, Not an Aesthetic Choice

**The default answer to "should we add this visual flourish" is no.** Every exception to this default (e.g., the single accent-blue color, the confidence-bar visualization in the Diagnosis Panel) is justified elsewhere in this document by a specific comprehension benefit — not simply because it looks more polished.

---

## 2. Color Palette (Exhaustive, With Every Usage Enumerated)

### 2.1 The Complete Palette

| Role | Color Name | Hex | Exact RGB |
|---|---|---|---|
| Background | Off-white | `#FAFAFA` | `250, 250, 250` |
| Surface | White | `#FFFFFF` | `255, 255, 255` |
| Border / Divider | Light gray | `#E5E5E5` | `229, 229, 229` |
| Primary text | Near-black | `#1A1A1A` | `26, 26, 26` |
| Secondary text | Mid gray | `#6B6B6B` | `107, 107, 107` |
| Accent (brand/interactive) | Deep blue | `#2451B8` | `36, 81, 184` |
| Accent hover state | Darker blue | `#1D3F94` | `29, 63, 148` |
| Success / Up | Green | `#1F9254` | `31, 146, 84` |
| Warning / Degraded | Amber | `#B8842A` | `184, 132, 42` |
| Danger / Down | Red | `#C4362F` | `196, 54, 47` |
| Danger hover state | Darker red | `#A32D27` | `163, 45, 39` |

### 2.2 Every Single Place Each Color Is Used — a Complete Enumeration

**`#FAFAFA` (Background):**
- The `<body>` background on every page.
- The sidebar's background (very slightly distinguished from the main content area, which sits on `#FFFFFF` cards).

**`#FFFFFF` (Surface):**
- Every card (endpoint rows, incident cards, stat blocks, the Diagnosis Panel container).
- Modal/dialog backgrounds (Add Endpoint form, confirmation dialogs).
- Table backgrounds.
- Input field backgrounds.

**`#E5E5E5` (Border/Divider):**
- The 1px border around every card.
- Horizontal rules between table rows.
- Input field borders (in their default, unfocused, non-error state).
- The vertical divider between the sidebar and main content, if a visual separator is used at all (a difference in background color between `#FAFAFA` and `#FFFFFF` may be sufficient on its own without an explicit border line — implementer's judgment, but only one or the other, never both, to avoid double-emphasizing the same boundary).

**`#1A1A1A` (Primary text):**
- Page titles, section headings.
- Endpoint names, incident titles.
- Table cell data (timestamps' actual values, status codes, response times).
- The Diagnosis Panel's explanation paragraph and suggested-next-step text.
- Form input values as the user types them.

**`#6B6B6B` (Secondary text):**
- Timestamps shown as metadata (e.g., "3 minutes ago" beside a table row).
- Form field labels.
- Placeholder text inside empty inputs.
- The "Diagnosis pending..." status line.
- Helper/caption text beneath form fields (e.g., password requirements).
- Chart axis labels.

**`#2451B8` (Accent):**
- Primary button backgrounds (Add Endpoint, Save, Run Evaluation, Log In, Create Account).
- Active/selected sidebar navigation item (as a left-border indicator plus bold text, not a filled background — Section 4.3 below).
- Links (e.g., "Fix and Retry," the "None of these" style secondary text-links, "Try logging in instead").
- Focus rings on inputs and buttons when navigating via keyboard (accessibility requirement, Section 8).
- The response-time graph's line.
- Confidence bars in the Diagnosis Panel's ranked-causes visualization.
- Bars in the Evaluation Results latency/cost comparison charts.

**`#1D3F94` (Accent hover):**
- Applied only on `:hover` and `:active` states of anything using `#2451B8` as its base color (primary buttons, links) — never used as a resting-state color anywhere.

**`#1F9254` (Success/Up):**
- The "Up" status badge's dot and text.
- The left border of a resolved Incident Card (note: this is a deliberate exception worth flagging — see Section 2.3 below for why resolved incidents actually use gray, not green, correcting an inconsistency from an earlier draft of this document).
- A brief, subtle success confirmation (e.g., a checkmark) after a setting saves successfully, if any such micro-confirmation is used at all — even this is optional given the "no color without status meaning" principle; a plain, wordless UI change (the toggle simply moving) is equally acceptable and arguably more consistent with minimalism.

**`#B8842A` (Warning/Degraded):**
- The "Degraded" status badge's dot and text, used specifically when an endpoint's checks are succeeding (correct status code) but response time exceeds the configured latency threshold.
- Nowhere else — amber is never used for any other purpose in the interface, keeping its meaning unambiguous.

**`#C4362F` (Danger/Down):**
- The "Down" status badge's dot and text.
- The left border of an open (unresolved) Incident Card.
- Destructive button borders/text ("Delete Endpoint").
- Inline validation error text and error-state input borders.
- Failed-check markers on the response-time graph (small dots).

**`#A32D27` (Danger hover):**
- Applied only on `:hover`/`:active` states of destructive buttons.

### 2.3 Correcting and Clarifying the Resolved-Incident Color

To be fully explicit (since this is a place earlier design thinking could plausibly be misread): a **resolved** incident's Incident Card uses a **mid-gray** left border (`#6B6B6B` or a slightly lighter neutral gray, e.g. `#9CA3AF`, if more visual distinction from body text is desired), **not** green. The reasoning: green is reserved exclusively for "this is currently healthy, right now" — an endpoint's live status. A resolved incident is a *historical record of a past problem*, which is a fundamentally different concept from "currently up," and reusing the same green would blur that distinction, potentially making a list of past incidents look deceptively reassuring ("look, everything's green!") when it's actually a history of things that broke. Gray communicates "this is closed/inactive/historical" without implying anything positive about current state.

### 2.4 The Hard Rule, Restated Precisely

Exactly **four** meaningful, status-or-interaction-bearing colors exist in the entire application: green, amber, red, and blue (plus each one's single hover-state variant, which is not a "new" color but a shade of the same one). Every other color used anywhere is drawn from the neutral scale (`#FAFAFA`, `#FFFFFF`, `#E5E5E5`, `#6B6B6B`, `#1A1A1A`). No sixth color — no purple for "informational," no yellow distinct from the amber warning color, no teal accent — is ever introduced, including inside charts, including inside any future screen not explicitly covered in this document. If a future feature seems to need a new color, the correct response is to first ask whether it can be expressed using the existing four-plus-neutral system before adding anything new.

---

## 3. Typography (Exhaustive)

### 3.1 Font Stack

```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
```

No web font file is loaded. This is a deliberate performance and philosophy choice: it means every page's text renders immediately using whatever native system font is already available on the user's device, with zero additional network request and zero flash-of-unstyled-text risk — appropriate for a tool whose value is speed-of-comprehension, not brand-distinctive typography.

### 3.2 Complete Type Scale

| Token Name | Size | Weight | Line Height | Color | Used For |
|---|---|---|---|---|---|
| `heading-page` | 24px | 600 (semi-bold) | 1.2 | `#1A1A1A` | Page titles ("Dashboard," "Incident Detail," "Settings") |
| `heading-section` | 18px | 600 | 1.2 | `#1A1A1A` | Section headings within a page ("Recent Checks," "Diagnosis," "Response Time — Last 24 Hours") |
| `body` | 14px | 400 (regular) | 1.5 | `#1A1A1A` | Table cell data, form field values, explanation paragraphs, button labels |
| `body-emphasis` | 14px | 600 | 1.5 | `#1A1A1A` | Endpoint names within rows/cards, incident card titles — a semi-bold weight at body size, used to create hierarchy without changing size |
| `caption` | 12px | 400 | 1.5 | `#6B6B6B` | Timestamps, field labels, helper text, chart axis labels, "Diagnosis pending..." text |
| `caption-link` | 12px | 400 | 1.5 | `#2451B8` | Small inline links ("Fix and Retry," "Add custom headers") |

### 3.3 Weight Restriction, Stated Precisely

Only two numeric font weights are used anywhere in the application: **400 (regular)** and **600 (semi-bold)**. No 300 (light), no 500 (medium), no 700+ (bold) or 900 (black) weight is ever used. No italic style is used anywhere — not even for emphasis within body text (use semi-bold for emphasis instead, per `body-emphasis` above, keeping the "no decorative typographic flourish" principle consistent even at the level of individual words within a sentence).

### 3.4 A Worked Typographic Example — the Incident Card

To make the type scale concrete, here's exactly how one specific, real component uses it:

- "Payments API — Production" (the endpoint name) → `body-emphasis` (14px, 600, `#1A1A1A`)
- "Ongoing — started 14:02" (the subtitle) → `caption` (12px, 400, `#6B6B6B`)
- "Likely cause: DB connection pool exhaustion" (the diagnosis summary line) → `body` (14px, 400, `#1A1A1A`) — deliberately body-weight, not caption-weight, because this is meaningful content the user needs to actually read and register, not peripheral metadata like the timestamp above it

---

## 4. Layout System (Exhaustive)

### 4.1 Grid and Content Width

- Maximum content width: **1200px**, horizontally centered on any viewport wider than that.
- Side padding on narrower viewports: **24px** minimum on each side, so content never touches the screen edge.
- A 12-column grid underlies multi-column layouts (e.g., dashboard stat cards arranged 3-or-4-across on desktop), though many screens (forms, the Diagnosis Panel) are intentionally single-column regardless of available width, since forcing extra columns where there's no content need would only add unnecessary complexity.

### 4.2 Spacing Scale, Stated as a Hard Constraint

The only permitted spacing values anywhere in the application — margins, paddings, gaps between elements — are: **4px, 8px, 16px, 24px, 32px, 48px.** No value outside this set (e.g., 12px, 20px, 30px) is used anywhere. This is deliberately restrictive: it means every implementer, working on any screen, is choosing from the same small palette of spacing decisions, which is what produces the felt consistency described in Principle 1.4 — not a stylistic accident, but a structural constraint applied everywhere.

**Worked example — spacing within the Diagnosis Panel:**
- 24px padding around the entire panel's contents.
- 16px vertical gap between the engine tag and the ranked-causes list.
- 8px vertical gap between each individual ranked-cause bar.
- 24px vertical gap between the ranked-causes list and the explanation paragraph.
- 16px vertical gap between the explanation paragraph and the suggested-next-step line.

### 4.3 Navigation — Exact Behavior

**Desktop (≥1024px):** a fixed-width (approximately 220px) left sidebar, background `#FAFAFA`, containing: the RootMind name/logo (text-only, no icon logo needed — simplicity), then four nav items (Dashboard, Incidents, Evaluation, Settings), each 40px tall, with 16px horizontal padding. The **active** item is distinguished by: a 3px accent-blue left border flush against the sidebar's edge, plus the item's text rendered in `body-emphasis` (semi-bold) instead of `body` (regular) weight — deliberately **not** a filled/colored background block behind the active item, since a filled background reads as visually heavier and slightly more "designed" than this project's restrained direction calls for.

**Tablet/Mobile (<1024px):** the sidebar collapses entirely into a top bar (approximately 56px tall, background `#FFFFFF`, bottom border `#E5E5E5`) containing the RootMind name on the left and a hamburger icon on the right. Tapping the hamburger reveals a slide-in panel (sliding in from the left, covering roughly 75% of the screen width, with a semi-transparent dark overlay over the remaining visible content) containing the same four nav items, laid out identically to the desktop sidebar's item styling.

### 4.4 Cards — the Single Reusable Container

Every card in the application — regardless of what it contains — shares exactly this treatment:
- Background: `#FFFFFF`
- Border: 1px solid `#E5E5E5`
- Border radius: 8px
- Padding: 16px for compact cards (endpoint list rows), 24px for content-heavier cards (the Diagnosis Panel, stat blocks)
- **No box-shadow, ever.** A card is distinguished from the page background purely by its white fill against the `#FAFAFA` page background plus its thin border — not by an elevation shadow, which would introduce a sense of depth/layering this flat, minimalist design language deliberately avoids.

---

## 5. Component Guidelines — Every Component, Every State

### 5.1 Status Badge

**Anatomy:** an 8px-diameter colored circle, followed by 6px of horizontal spacing, followed by the status word in `caption`-scale text (12px) but colored to match its status (not the default secondary-gray) — i.e., a green dot plus green "Up" text, a red dot plus red "Down" text, an amber dot plus amber "Degraded" text.

**Why the text itself is colored, not just the dot:** this is a specific, deliberate reinforcement of the "never color alone" accessibility principle — even someone who cannot distinguish the dot's color at all still sees the literal word "Down," and even someone who can see color but is scanning quickly gets a stronger, doubled signal from both the dot and the colored word together.

**All three states, fully specified:**
- **Up:** green dot, green text, reads "Up"
- **Down:** red dot, red text, reads "Down"
- **Degraded:** amber dot, amber text, reads "Degraded"

**A fourth, non-status state worth naming explicitly:** a brand-new endpoint that hasn't had its first scheduled check yet (per App Flow, Flow A.2, step 7) shows a **gray** dot (using the secondary-text gray, `#6B6B6B`, not any of the three status colors, since it isn't actually a status yet) with the text "Pending first check" — this is neither Up, Down, nor Degraded, and must not borrow any of those three colors, since doing so would misleadingly imply a status has already been determined.

### 5.2 Endpoint List Row (Dashboard)

**Layout:** a single horizontal row inside a card-style container (per Section 4.4's card spec, using the 16px "compact" padding), split into two halves — left-aligned content (endpoint name in `body-emphasis`, immediately followed by the Status Badge with 12px of spacing between name and badge) and right-aligned content (uptime percentage, then average response time, each in `caption` styling, separated by 16px).

**Interactive states:**
- **Default (resting):** as described above, background `#FFFFFF`.
- **Hover:** background transitions to `#F5F5F5` over 150ms using a simple `ease` timing function — no other property changes (no border color shift, no text color shift, no scale transform).
- **Focus (keyboard navigation):** a 2px accent-blue outline appears around the entire row, offset 2px from the row's edge, satisfying the keyboard-accessibility requirement (Section 8) without needing a separate visual treatment from hover.
- **Active/pressed (mouse down, before release):** background darkens very slightly further, to `#EFEFEF`, giving tactile feedback that a click was registered — released back to hover or default state on mouse-up depending on where the cursor ends up.

**Sort order (a behavioral, not purely visual, specification):** rows are sorted Down-status first, then Degraded-status, then Up-status, with ties within each group broken alphabetically by endpoint name — ensuring a stable, predictable order rather than one that visually jumps around unpredictably between refreshes when multiple endpoints share the same status.

### 5.3 Response Time Graph

**Specification:**
- A single line, 2px stroke width, color `#2451B8` (accent blue), with rounded line joins (no sharp mitered corners at data points, which would look more technical/jagged than the calm aesthetic calls for).
- No fill/gradient beneath the line.
- Gridlines: horizontal only (no vertical gridlines, which would add visual clutter without proportionate benefit for a graph that's primarily about the overall trend shape, not precise point-in-time lookup), 1px, color `#E5E5E5`, drawn at 4–5 evenly-spaced value intervals.
- Failed checks: rendered as small solid circles, 6px diameter, color `#C4362F` (danger red), placed exactly at their timestamp position along the x-axis, at a fixed low point near the x-axis itself (not at their "response time" y-position, since a failed check often has no meaningful response time to plot, e.g., a DNS failure) — visually reading as "something happened here" markers along the timeline rather than as data points within the line series itself.
- Axis labels: `caption` styling (12px, `#6B6B6B`).
- **A required accompanying text summary** (per the Accessibility section): one line, in `caption` styling, placed directly beneath the graph, e.g., "Average response time: 184ms over the last 24 hours" — computed from the same dataset driving the visual graph.

**Worked scenario — an endpoint with zero checks yet (brand new):** rather than rendering an empty/broken-looking chart axis with no line at all, the graph area shows the Empty State treatment (Section 5.7) with the text "No check data yet — your first check will run within a minute."

### 5.4 Incident Card (Incidents List)

**Layout:** a card (per Section 4.4, compact 16px padding) with a 4px-wide colored left border flush against the card's left edge — red (`#C4362F`) for `status = "open"`, gray (`#9CA3AF`, per the clarification in Section 2.3) for `status = "resolved"`.

**Content, top to bottom within the card:**
1. Endpoint name, `body-emphasis` (14px, 600).
2. Subtitle line, `caption` (12px, `#6B6B6B`): for open incidents, "Ongoing — started [time]"; for resolved incidents, "Resolved — lasted [duration], ended [time]".
3. Diagnosis summary line, `body` (14px, 400, `#1A1A1A`) — deliberately full body weight/color, not caption styling, since per Section 3.4's reasoning this is meaningful content: either "Likely cause: [top cause name]" once available, or "Diagnosis pending..." in `caption` styling specifically while genuinely pending (the one case where this third line does use the lighter caption treatment, since "pending" is transient status metadata, not yet real content).

### 5.5 Diagnosis Panel (Incident Detail Page)

**Full anatomy, top to bottom:**

1. **Engine tag** — a small pill, background `#F5F5F5` (a slightly darker neutral than the page background, to read as a distinct, contained label), text `caption` styling but in `#1A1A1A` (not secondary-gray, since the engine name is meaningful metadata worth reading clearly) — e.g., "Prompt-based LLM."
2. **Ranked causes list** — each cause is one row: cause name in `body` styling on the left, a horizontal bar (accent blue `#2451B8`, height 8px, rounded ends) whose width is proportional to its confidence percentage, and the percentage itself in `caption` styling on the right, right-aligned. Causes are always in descending confidence order, top to bottom.
   - **Worked scenario — a very long cause name** (e.g., "Database connection pool exhaustion due to sustained high concurrent request volume"): the bar visualization's width is fixed and independent of text length (it's driven purely by the confidence percentage value, never by text length), and the cause name text wraps naturally onto a second line if needed, with the bar and percentage remaining aligned to the first line's baseline — the layout must never let a long cause name distort or compress the bar's proportional meaning.
   - **Worked scenario — many ranked causes returned** (e.g., an engine returns 5 causes instead of the typical 3): all are shown, not truncated — there is no hardcoded maximum in the UI, since arbitrarily hiding lower-ranked causes could hide useful information, and the vertical space cost of a few extra rows is low compared to the cost of hiding potentially relevant information.
3. **Explanation paragraph** — `body` styling, full width of the panel, normal paragraph wrapping, no truncation, no "read more" interaction, regardless of length.
   - **Worked scenario — an unusually long or verbose explanation** from a given engine: it is shown in full. If this becomes visually overwhelming in practice during implementation, the correct fix is improving the prompt/model output to be more concise (a content-generation fix), never truncating the display (a UI workaround that would hide potentially important reasoning).
4. **Suggested next step** — visually set apart via 16px of additional top spacing plus a short bold-weight label "Suggested next step:" (using `body-emphasis` for just that label phrase) immediately preceding the plain `body`-styled suggestion text, so it reads as a distinct, actionable callout rather than blending into the explanation paragraph above it.
5. **Re-run diagnosis control** — a `caption`-scale dropdown (engine selector) plus a secondary-style button reading "Run," both visually smaller and positioned with extra top spacing (24px) below the main panel content, deliberately de-emphasized relative to the diagnosis content itself.

**Worked scenario — `status = "pending"`:** the entire panel (steps 1–4 above) is replaced with a single centered line of `caption`-styled text reading "Generating diagnosis..." — no spinner icon or loading animation is used here (consistent with the "no unnecessary motion" principle); the text itself, combined with the page's automatic polling (App Flow, Part D.1), is sufficient.

**Worked scenario — `status = "unavailable"`:** the panel shows the engine tag (so it's clear which engine was attempted) followed by a single line: "Diagnosis unavailable for this attempt." in `body` styling (not styled as an alarming error — this is a known, gracefully-handled outcome, not a system failure the user needs to worry about), plus a secondary-style "Retry" button.

### 5.6 Buttons — Every Variant and State

| Variant | Background | Text Color | Border | Used For |
|---|---|---|---|---|
| Primary | `#2451B8` (hover: `#1D3F94`) | `#FFFFFF` | none | Add Endpoint, Save, Log In, Create Account, Run Evaluation |
| Secondary | `#FFFFFF` | `#2451B8` | 1px solid `#2451B8` | Cancel, Re-run Diagnosis, Retry, Fix and Retry |
| Destructive | `#FFFFFF` (hover: fills to `#A32D27` with white text) | `#C4362F` (hover: `#FFFFFF`) | 1px solid `#C4362F` | Delete Endpoint |
| Disabled (any variant) | `#E5E5E5` | `#9CA3AF` | none | Any button mid-submit (e.g., "Testing connection..." during Add Endpoint), or any action correctly unavailable given current state |

**Sizing:** all buttons share a consistent height of 40px, horizontal padding of 16px, border-radius of 6px, and `body`-scale (14px) semi-bold text — regardless of variant, so a Primary and a Secondary button placed side by side (e.g., "Save" and "Cancel" in a form) align perfectly.

**Worked scenario — the destructive button's hover state, spelled out precisely:** unlike Primary and Secondary, which only shift their background/border shade slightly on hover, the Destructive button's hover state is a full color inversion (from white-background-red-text to red-background-white-text) — a deliberately slightly stronger visual reaction than other buttons receive, appropriate given that hovering signals imminent intent to perform a destructive, harder-to-reverse action, and a slightly more pronounced visual acknowledgment is warranted.

### 5.7 Empty States — Every Instance Enumerated

| Location | Icon Concept | Message | Call to Action |
|---|---|---|---|
| Dashboard, zero endpoints | A simple line-art "signal/pulse" glyph | "You haven't added any endpoints yet." | "Add Your First Endpoint" button (Primary) |
| Endpoint Detail, zero checks yet | A simple line-art clock/hourglass glyph | "No check data yet — your first check will run within a minute." | None (no action needed; this resolves itself automatically) |
| Endpoint Detail, zero incidents | A simple line-art checkmark/shield glyph | "No incidents recorded for this endpoint yet — that's a good sign." | None |
| Incidents List, filtered to "Open," zero results | A simple line-art checkmark/shield glyph | "No open incidents. Everything's healthy." | None |
| Incidents List, filtered to "Resolved," zero results | Same checkmark/shield glyph | "No resolved incidents yet." | None |
| Evaluation Results, no run has ever completed | A simple line-art bar-chart glyph | "No evaluation results yet. Run a comparison to see how the three diagnosis engines perform." | "Run Comparative Evaluation" button (Primary) |

**Consistent structure across all of the above:** icon (48px, single-color `#6B6B6B` line-art, never colored decoratively), 16px spacing, message text in `body` styling (`#1A1A1A`, not secondary-gray — the message itself is meaningful content worth reading clearly, even though it's describing an absence), 16px spacing, then the call-to-action button if one applies, or nothing further if the empty state is simply informational (e.g., "everything's healthy" needs no button, since there's no action to take).

### 5.8 Forms — Every Field State

**Default (untouched) state:** label above field in `caption` styling; input with 1px `#E5E5E5` border, `#FFFFFF` background, `body`-scale text.

**Focused state:** border changes to 2px `#2451B8` (accent blue) — a slightly thicker border on focus rather than just a color change, to make the focus state clearly perceptible even for users who may have difficulty distinguishing the blue from the default gray by color alone.

**Valid, filled state (after successful blur validation):** border returns to the default 1px `#E5E5E5` — validity is not specially celebrated with a green border or checkmark (consistent with reserving green strictly for endpoint-up status), since a correctly-filled form field is the expected, unremarkable default outcome, not something requiring positive reinforcement.

**Invalid state (after failed blur validation, or failed server-side validation on submit):** border changes to 1px `#C4362F` (danger red), and an inline error message appears directly beneath the field in `caption`-scale red text (`#C4362F`), e.g., "Please enter a valid URL."

**Disabled state:** background `#FAFAFA` (matching the page background, visually receding), text `#6B6B6B`, border `#E5E5E5`, cursor shows "not-allowed" — used, for example, on the Email field during the brief moment a login submission is in flight.

---

## 6. What This Design Explicitly Avoids, With Fuller Reasoning

| Avoided Pattern | Full Reasoning |
|---|---|
| Dark mode as a launch requirement | Implementing dark mode properly requires an entire second, equally rigorous palette (every color in Section 2 would need a dark-mode counterpart chosen with the same contrast-ratio discipline), which is a meaningful design and QA effort disproportionate to this project's timeline and grading priorities — explicitly deferred as an optional stretch goal only, never a blocking requirement |
| Draggable/rearrangeable dashboard widgets | Beyond the added interaction-engineering complexity, a fixed, always-in-the-same-place layout is actually *faster* to scan precisely because the user's eye learns exactly where to look over repeated use — customizability here would work against the core "see status fast" goal, not just add unnecessary effort |
| Decorative illustrations or a mascot | Directly conflicts with the "utility tool, not a consumer brand" positioning stated in the Design Philosophy; also, illustration work is genuinely time-consuming to do well, and doing it poorly (generic stock-style icons) would look worse than having no illustration at all |
| Skeuomorphic effects (drop shadows, gradients, glassmorphism, glow effects) | Every one of these effects implies a sense of physical depth/layering — cards "floating" above the page, buttons that look physically raised — which is a deliberately different visual metaphor than the flat, calm, paper-like feel this design intentionally pursues |
| A second accent color | Introducing a second accent color would immediately create ambiguity: does this new color mean something (a new status?) or is it purely decorative? The single-accent-color rule (Section 2.4) eliminates that ambiguity structurally, for every future addition to the app, not just the screens specified today |
| Bouncy, springy, or attention-seeking animation | Any animation with overshoot/bounce easing reads as playful or attention-grabbing — both are wrong tones for a tool that may be open during a genuine, stressful outage; the only animation permitted anywhere is a simple linear-or-ease opacity/background-color fade, capped at 200ms |
| Toast/snackbar notifications for routine confirmations | Per Section 5.8 and the Settings flow (App Flow, Flow A.9), routine successful actions (a setting saved, a form submitted successfully) are confirmed by the UI's own state change (the toggle moved, the modal closed and the new row appeared) rather than by a separate transient notification — an extra notification layer is unnecessary machinery when the state change itself is already sufficient confirmation |

---

## 7. Responsive Behavior — Every Breakpoint, Every Screen Type

| Breakpoint | Navigation (Section 4.3) | Tables | Cards | Forms |
|---|---|---|---|---|
| Desktop (≥1024px) | Full sidebar | Standard multi-column tables | Multi-column grids where applicable (e.g., dashboard stat summary blocks, if used) | Single-column, comfortable width (~480px), centered within available space |
| Tablet (768–1023px) | Top bar + hamburger | Tables remain tables, with reduced horizontal cell padding (16px → 8px) to fit more comfortably | Single-column card stacks | Single-column, full available width minus 24px side padding |
| Mobile (<768px) | Top bar + hamburger | **Tables convert to stacked card lists** — each row becomes its own compact card, with each column's label and value shown as a stacked label-then-value pair rather than side-by-side columns (e.g., instead of a row reading "14:02 | 200 | 184ms," a mobile card shows three stacked lines: "Time: 14:02," "Status: 200," "Response time: 184ms") | Single-column card stacks | Single-column, full width minus 16px side padding (slightly tighter than tablet, given the smaller available space) |

**Why the mobile table-to-card conversion matters enough to specify in this much detail:** a horizontally-scrolling table on a narrow phone screen is one of the most common, well-documented sources of mobile usability frustration — a user has to scroll sideways to see columns that got cut off, often without a clear visual cue that more content exists off-screen. Converting to a stacked-card format guarantees every piece of data is immediately visible without any horizontal scrolling, directly serving the "read status fast" principle even on the smallest supported screens.

---

## 8. Accessibility — Every Requirement, With the Reasoning Behind Each

### 8.1 Contrast Ratios (Specific, Checked Pairings)

| Foreground | Background | Approximate Contrast Ratio | Meets 4.5:1? |
|---|---|---|---|
| `#1A1A1A` (primary text) | `#FFFFFF` (surface) | ~16.1:1 | Yes, comfortably |
| `#1A1A1A` (primary text) | `#FAFAFA` (page background) | ~15.6:1 | Yes, comfortably |
| `#6B6B6B` (secondary text) | `#FFFFFF` | ~5.2:1 | Yes |
| `#2451B8` (accent blue) | `#FFFFFF` | ~6.3:1 | Yes |
| `#1F9254` (success green) | `#FFFFFF` | ~4.6:1 | Yes, though close to the threshold — this pairing should be re-verified with an actual contrast-checking tool once the final font weight/size for badge text is settled, since 4.5:1 is a minimum, not a comfortable margin |
| `#C4362F` (danger red) | `#FFFFFF` | ~5.9:1 | Yes |
| `#B8842A` (warning amber) | `#FFFFFF` | ~4.5:1 | Borderline — this is the pairing most worth double-checking with a real tool during implementation, and adjusting slightly darker if a precise measurement comes in under 4.5:1 |

**Action item flagged for implementation:** the success-green and warning-amber pairings sit close enough to the 4.5:1 minimum that they should be verified with an actual browser-based contrast checker (not just this document's approximate calculation) once real font rendering is in place, and darkened slightly if needed.

### 8.2 Never Color-Alone — Every Place This Principle Is Actively Applied

- Status badges: color dot **plus** colored text (Section 5.1).
- Incident Card left borders: color **plus** the "Ongoing"/"Resolved" text in the subtitle line — a user relying purely on the text, with no color perception at all, still gets the same information.
- Form validation: red border **plus** an explicit written error message — never a bare red outline with no accompanying text explaining what's wrong.
- Failed checks on the response-time graph: red dots **plus** the fact that they're already distinguishable by their position (sitting near the x-axis baseline, distinct from the response-time line's data points) — a secondary, non-color-dependent visual cue.

### 8.3 Keyboard Navigation

Every interactive element — sidebar nav items, buttons, form inputs, entire clickable card/row surfaces (like the Endpoint List Row) — must be a real focusable element (a proper `<button>`, `<a>`, or an element with `tabindex="0"` plus appropriate `role` and keyboard event handling if a non-native element is used for a clickable row) reachable via `Tab`, in the same left-to-right, top-to-bottom order a sighted user would visually scan the page. `Enter` or `Space` activates any focused button-like element, matching native browser button behavior exactly, even for custom-styled elements.

### 8.4 Chart Accessibility — the Required Text Summary Pattern

Every chart in the application (the response-time graph; the Evaluation Results latency and cost comparison bar charts) is required to include a one-line, plain-text summary directly beneath or beside it, computed from the same data driving the visual — for example: "Average response time: 184ms over the last 24 hours" or "Engine 1 achieved the highest top-1 accuracy at 78%, followed by Engine 2 at 71%." This serves two overlapping purposes worth naming both of: it makes the chart's key takeaway accessible to screen-reader users who cannot perceive the visual chart at all, and it lets any sighted user (including a mentor quickly scanning during a fast-paced viva) get the headline number immediately without needing to visually interpret bar heights or line slopes themselves.

---

## 9. Design Consistency Checklist — Expanded, for Use Before Marking Any Screen "Done"

- [ ] Uses only colors from the defined palette (Section 2) — no new hex values introduced anywhere, including in inline styles or one-off CSS
- [ ] Uses only the defined type scale tokens (Section 3.2) — no ad-hoc font sizes, no font-weight values other than 400 or 600
- [ ] All spacing values are drawn from the approved 4/8/16/24/32/48px scale (Section 4.2) — verified by inspecting the actual CSS, not just eyeballing the rendered result
- [ ] Any card-like container uses the standard card primitive exactly (Section 4.4: white surface, 1px `#E5E5E5` border, 8px radius, no box-shadow)
- [ ] Any status is shown with both color and text, never color alone (Section 8.2) — verified by checking each instance against the enumerated list in that section
- [ ] No animation exceeds 200ms or uses anything beyond a simple linear/ease fade — no bounce, spring, or scale-transform animations exist anywhere
- [ ] The screen has been manually checked at all three responsive breakpoints (Section 7), including verifying that any table correctly converts to the stacked-card format below 768px
- [ ] Any chart includes a plain-text summary line directly beneath or beside it (Section 8.4)
- [ ] Every interactive element has been tab-tested for keyboard reachability and correct focus-order (Section 8.3)
- [ ] Every button on the screen matches one of the four defined variants exactly (Section 5.6) — no one-off button styling introduced for a specific screen's "special" action
- [ ] Every empty-state instance on the screen (if any apply) matches the structure and tone defined in Section 5.7 — icon, message, optional single call-to-action, nothing more elaborate
