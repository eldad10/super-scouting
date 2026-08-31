# FRC Scouting Platform — Design Specification (Living Document)

**Status:** DRAFT v0.24 — **topics 1–13 CLOSED** (every feature topic). Core architecture forks decided (storage, versioning, offline stats, stack, access model, language, roles/auth, data-entry UX, offline-first & sync, realtime deferred → 45s refresh, dashboards & visualisation, search/ranking/browse, data quality, alliance selection). Remaining: 14 (formal close of the deferred TBA import), 18, 19, and the partials 15/16/17/20.
**Last updated:** 2026-08-31
**Owner:** (your name / team number)
**Destination:** this document, once all topics are CLOSED, becomes the input spec for Claude Code to generate an implementation plan.

---

## 0. How to use this document

### 0.1 Structure

The document is split into **topics** (sections 2–19). Every topic contains:

- **Confirmed requirements** — things you already told me, restated precisely. These are locked unless you change them.
- **Proposed decisions** — my recommendation where a decision is needed, with the alternatives and trade-offs. Nothing here is locked until you say so.
- **Open questions** — numbered `Q<section>.<n>` so we can reference them in chat (e.g. "Q4.3: option B").

### 0.2 Topic status

| # | Topic | Status |
|---|-------|--------|
| 1 | Product vision & scope | CLOSED |
| 2 | FRC domain model & glossary | CLOSED |
| 3 | Dynamic forms — the core architecture decision | CLOSED |
| 4 | Seasons, competitions & events | CLOSED |
| 5 | Users, roles, authentication & permissions | CLOSED |
| 6 | Scouting data entry (runtime UX) | CLOSED |
| 7 | Offline-first & synchronisation | CLOSED |
| 8 | Realtime & live updates | CLOSED — realtime → §24; v1 uses 45s refresh |
| 9 | Statistics engine & computed metrics | CLOSED |
| 10 | Dynamic dashboards, graphs & visualisations | CLOSED |
| 11 | Search, ranking & browse | CLOSED |
| 12 | Alliance selection & pick list | CLOSED |
| 13 | Data quality, integrity & scouter reliability | CLOSED |
| 14 | External data integration (TBA / FRC Events API) | OPEN |
| 15 | System architecture, repo layout & services | PARTIAL — Q15.1/Q15.2/Q15.4/Q15.8 closed |
| 16 | UI/UX design system & responsive behaviour | PARTIAL — Q16.2 closed |
| 17 | Non-functional requirements | PARTIAL — Q17.3 closed (free tier) |
| 18 | Deployment, environments & operations | OPEN |
| 19 | Delivery phases & priorities | OPEN |
| 20 | AI insights, LLM integration & MCP readiness | SCOPED — setup only, Q20.5/Q20.6 open |

### 0.3 Working agreement

- We close topics one at a time. When a topic is closed I move its decisions from "proposed" to "confirmed", set the status to **CLOSED**, and log it in section 20.
- If I think you are forgetting something important, I will raise it whether or not you asked. Items I raise on my own initiative are marked **[RAISED BY ME]**.
- I will not write application code until the spec is approved and an implementation plan exists.

---

## 1. Product vision & scope

### 1.1 Confirmed requirements

- A **scouting platform for an FRC team** that is **year-agnostic**: the app must never need a code change to support a new game season. Everything game-specific is configuration created inside the app.
- The team can **create, edit and delete the data-collection forms** (which act as the "tables" representing each year's game) from inside the app UI.
- Data is **organised by season and event**. **Forms belong to a season** (a season may hold several — match / super / …), and every event in that season uses them (Q3.6). *(amended 2026-08-17)*
- Users can **search and analyse data per competition**, and also **aggregated across all competitions in the same year**.
- Users can build **statistics and visualisation pages dynamically from the UI**, choosing chart types and pointing at specific form fields.
- Separate **client** and **server** projects, in **one repository**, but as **two independently deployed services**.
- **Offline capable**: at a venue with no internet the app loads existing data, allows changes, and syncs when connectivity returns.
- **Responsive**: must look and work well on both phone and desktop.
- Database: **Supabase**.
- **Optimistic local UI**: a user's own action appears instantly on their own device with no manual refresh (except while offline). *(confirmed 2026-08-14)*
- **Cross-device live push** (another device's changes appearing without a refresh) is **deferred to nice-to-have** — see topic 8 and §24 (Q15.1).
- Deployment: **Vercel** for both client and server.
- Features to include: team search, form management, ranking, statistics (details per topic below).
- **Single-team install.** Built only for our team; multi-tenant/multi-org is out of scope (Q1.4). *(confirmed 2026-08-14)*
- **Team & events.** This is **FRC team 2096**, competing in the **FIRST Israel district** plus the **FIRST Championship**. The event model covers district events, championship divisions, and internal scrimmages — **each as a regular flat event, no division hierarchy** (Q4.3, resolved 2026-08-17). *(confirmed 2026-08-15, Q1.2)*
- **Scale at a competition.** Peak concurrent use is roughly **8 scouters + 2 strategy leads + 1 admin (~11 people)**. This is small — it confirms the JSONB storage model (§3) and the all-through-server access model (§15) are safe, and keeps realtime/scale targets modest (§17). *(confirmed 2026-08-15, Q1.1)*
- **Replaces a prior app** that failed on exactly the two headline requirements: it **did not work offline** and was **not versatile across seasons**. Offline-first (§7) and year-agnostic forms (§3) are therefore the primary, non-negotiable design drivers, validated by lived pain. *(confirmed 2026-08-15, Q1.3)*
- **Low-maintenance, multi-season lifespan.** Maintained by a **team mentor**, not a rotating student. The system must run season after season with **minimal intervention** ("if it works, nobody should need to touch it"). This favours boring, well-documented technology over clever solutions, and makes a **maintenance/handover checklist a v1 deliverable** — what to check, renew (keys, tokens, plan limits) and update at the start of each season and when the maintainer eventually leaves. Connects to Q18.5 (account ownership survives handover). *(confirmed 2026-08-15, Q1.6)*
- **Target completion 2026-10-01** (before the season ramps up). A soft target, not a hard deadline. *(confirmed 2026-08-15, Q1.5)*

### 1.2 Confirmed decisions

*(Confirmed 2026-08-15 on closing topic 1.)*

**Primary success criterion.** *"During a competition, the team's scouters can each record their assigned robot per match on a phone with no internet, and within 60 seconds of regaining connectivity the strategy lead sees updated team rankings on a laptop."* If that works, the product works.

**Explicit non-goals for v1:**

- Not a public/multi-tenant SaaS for other teams (single-team install — Q1.4 closed).
- No native iOS/Android app — a PWA installable to the home screen.
- No live-video or camera-based automatic scouting.
- No integration with the FMS/field systems.

### 1.3 Open questions

- ~~**Q1.1** — How many people use this at a competition, roughly?~~ ✓ **CLOSED 2026-08-15: ~8 scouters + 2 leads + 1 admin (~11 peak)** (§1.1).
- ~~**Q1.2** — What is your **team number and district/region**?~~ ✓ **CLOSED 2026-08-15: team 2096, FIRST Israel district + FIRST Championship (with divisions)** (§1.1).
- ~~**Q1.3** — What are you using today, and what fails about it?~~ ✓ **CLOSED 2026-08-15: a prior app that failed offline and wasn't year-agnostic — the two primary design drivers** (§1.1).
- ~~**Q1.4** **[RAISED BY ME]** — Should the system support **multiple teams/organisations** in one deployment (multi-tenant), or is it strictly your team?~~ ✓ **CLOSED 2026-08-14: single team** (§1.1).
- ~~**Q1.5** — Is there a hard deadline?~~ ✓ **CLOSED 2026-08-15: soft target 2026-10-01, not hard** (§1.1).
- ~~**Q1.6** **[RAISED BY ME]** — Who maintains this after you?~~ ✓ **CLOSED 2026-08-15: a team mentor; must be low-maintenance and multi-season, with a handover/maintenance checklist as a v1 deliverable** (§1.1).

---

## 2. FRC domain model & glossary

### 2.1 Why this section exists **[RAISED BY ME]**

You said "year-agnostic", which is right — but there is a set of concepts that is *stable across every FRC season*. Modelling those as first-class entities (rather than as fields inside a dynamic form) is what makes the app genuinely reusable. If team number or match number lives inside the flexible JSON blob, then ranking, search and cross-season comparison all break. So we split the world in two:

**Fixed skeleton (hard-coded, same every year):**

| Entity | Meaning |
|---|---|
| Season | A year, e.g. 2026. Has one game and one **scoring model** (below). |
| Event / Competition | A specific competition — district event, championship division, or internal scrimmage — **each modelled as a regular, flat event**; identified by **name + year**; belongs to a season. No division hierarchy (Q4.3). |
| Team | An FRC team: **team number + team name only**. Global, spans seasons. |
| Match | A match at an event: type (practice / qualification / playoff), match number, the 3 red teams, the 3 blue teams, and a **reserved nullable official result** (score / RP / W-L). Only **qualification** matches feed metric aggregates (Q2.6). |
| Scouter (User) | A person entering data. |
| Scouting Entry | One observation: one scouter, one team, one match, one form version → a payload of answers. Covers all **6 robots per match, including team 2096's own** (Q2.3). |

**Flexible layer (created by you in the app, differs every year):**

| Entity | Meaning |
|---|---|
| Form | The game-specific questionnaire ("2026 match scouting"). |
| Form Version | An immutable snapshot of a form's fields. |
| Field | One question: type, label, validation, section, conditional logic. |
| Scoring model | Maps each field's raw value → game points (boolean → points if true, counter/number → points per unit, single/multi-select → points per selected option). **Points are non-negative and defined per field per phase** (a field may score differently in auto/teleop/endgame). A **mission is successful when its points > 0** (no separate predicate). A robot's scouted score is the **sum of its field points**; a match/alliance total is the sum across its robots (no alliance-level bonus/threshold points). Edited **inline in the form builder** (editing a field carries its scoring with it, keyed by field) but stored **separately from the form version and versioned on its own timeline** — a scoring change is **never** a new form version and never invalidates entries. Entries store **raw observations only**; the score is always **derived**, so a corrected model re-scores all history. Game-specific, **not backfillable**; the metric engine (topic 9) implements it. **[RAISED BY ME → confirmed]** |
| Metric | A computed value derived from fields and the scoring model (e.g. "total game pieces = auto + teleop"). |
| Dashboard / Chart | A saved visualisation configuration referencing fields and metrics. |

### 2.2 Confirmed requirements

- **Fixed/flexible split confirmed** (Q2.1). Fixed schema: Season, Event, Team, Match, Scouter, Scouting Entry — each entry carries structured foreign keys plus a flexible game payload. Event = **name + year**; Team = **number + name only**.
- **Team numbers are global and stable** — team 2096 in 2024 and 2026 is the same team, so multi-season history works for free.
- **Season scoring model** (Q2.1, Q2.8 — fully closed). A field's raw value maps to game points (boolean → points if true; counter/number → points per unit; single/multi-select → points per selected option). Rules: **points are non-negative** — penalties are recorded but never subtracted (Q2.8b); **points are defined per field per phase** so a field can score differently in auto/teleop/endgame (Q2.8a); a **mission counts as successful when its points > 0** (Q2.8c); a robot's scouted score is the **sum of its field points** and a match/alliance total is the sum across robots (Q2.8e); **no alliance-level bonus/threshold points** (Q2.8f). Points are **entered inline in the form builder** but held in a **scoring model versioned independently of the form version**: entries store **raw observations only** and the score is **derived**, so **a scoring change never creates a new form version** and never invalidates collected data (Q2.8d). Captured at field-creation time (same non-backfillable gotcha as semantic metadata, §3.3). Topic 9 implements the engine.
- **Form kinds in v1: Match scouting + Super scouting only** (Q2.2). Pit / human / other remain addable later through the `kind` field; not built in v1.

| Form kind | One record per | Purpose |
|---|---|---|
| Match scouting | (team, match) | Quantitative performance in a match. |
| Super scouting / qualitative | (team, match) or (team, event) | Driver skill, defence, penalties, breakdowns — the subjective read. |

- **Scouting coverage** (Q2.3): all **6 robots per match, including our own**; one match-scouter per robot. With ~8 scouters (topic 1), the ~2 beyond the six field robots do super scouting.
- **No per-alliance records** (Q2.4): everything is attributed per robot.
- **Official match results** (Q2.5): schema is **reserved now as nullable** (score / RP / W-L) so no migration is needed when TBA import (§24) lands; until populated the **UI hides official-result displays entirely — it must never render an empty score box.**
- **Match inclusion in metrics** (Q2.6): practice and playoff matches are stored under their event and viewable on the search/forms page, but are **excluded from all metric aggregates**; only **qualification** matches are computed.
- **Own robot = a regular robot** (Q2.7, Q2.9): team 2096's robot is scouted exactly like any other of the 6 (Q2.3); there is **no separate reliability/mechanism/notes log**.

### 2.3 Resolved & spun-out questions

- ~~**Q2.1**~~ ✓ **CLOSED:** fixed/flexible split accepted; Event = name+year, Team = number+name. Season **scoring model** added (§2.1–§2.2).
- ~~**Q2.2**~~ ✓ **CLOSED:** match + super scouting only in v1.
- ~~**Q2.3**~~ ✓ **CLOSED:** all 6 robots incl. our own, one scouter each; spare scouters super-scout.
- ~~**Q2.4**~~ ✓ **CLOSED:** no per-alliance data.
- ~~**Q2.5**~~ ✓ **CLOSED:** reserve nullable official-result schema; UI never shows empty score boxes.
- ~~**Q2.6**~~ ✓ **CLOSED:** practice + playoff stored & searchable, excluded from metrics; qualification only.
- ~~**Q2.7 / Q2.9**~~ ✓ **CLOSED:** our own robot is scouted as a regular robot — no separate tracking log.

**Q2.8 — scoring model.** ✓ **FULLY CLOSED 2026-08-17.** A field's raw value maps to points (boolean → points-if-true; counter/number → points-per-unit; single/multi-select → points-per-selected-option), entered inline in the builder but stored in a scoring model versioned independently of the form; entries hold raw observations and the score is derived. Topic 9 implements the engine.
- ~~**Q2.8a**~~ ✓ **CLOSED:** points are defined **per field per phase** — a field may score differently in auto/teleop/endgame. *(How this coexists with the single required `phase` tag per field is a form-builder detail for topic 3.)*
- ~~**Q2.8b**~~ ✓ **CLOSED:** **no negative points** — penalties/fouls are recorded but never subtracted from the score.
- ~~**Q2.8c**~~ ✓ **CLOSED:** a mission is **successful when its points > 0**; no separate predicate.
- ~~**Q2.8d**~~ ✓ **CLOSED:** a scoring change is an **in-place edit of the scoring model — never a new form version**; because scores are derived, a corrected model re-scores all history.
- ~~**Q2.8e**~~ ✓ **CLOSED:** a robot's scouted score = **sum of its field points**; match/alliance total = sum across robots; this is what reconciles against the reserved official result (Q2.5).
- ~~**Q2.8f**~~ ✓ **CLOSED:** **no** alliance-level bonus/threshold points (RP thresholds, coopertition) in the scouted score.

---

## 3. Dynamic forms — the core architecture decision

> This is the single most important decision in the project. Everything else — offline sync, statistics, graphs, performance — is downstream of it. I want to close this topic first.

### 3.1 Confirmed requirements

- Forms are created, edited and deleted from inside the app by an admin.
- A form represents the game for a given year and is linked to competitions.
- Form fields are the targets that charts and statistics point at.
- **Storage model: Option A — one `scouting_entries` table with a JSONB payload** (Q3.1). Per-form-version SQL views flatten the JSONB into typed columns for analytics; validators and casts are **generated from the field definitions**, not hand-written per form. *(confirmed 2026-08-14)*
- **Immutable form versioning** (Q3.2). A new version is created only by *structural* changes — adding, removing/deprecating, or retyping a field. Editing a field's **label** or **range/min-max** happens in place and does **not** create a version; a range change applies to new entries only and **never retroactively invalidates data already collected**. Field `key`s are permanent. *(confirmed 2026-08-14)*
- **Semantic field metadata captured at field-creation time** (Q3.9). `description`, `unit`, `phase` and `direction` are **required** per field; `category`, `expected_range`, `include_in_ai_context` are optional. It cannot be backfilled and it drives the generated validators/views, chart labels and default sort (and is the LLM data dictionary later). *(confirmed 2026-08-14)*
- **Field type catalogue** (Q3.3). All catalogue types ship **except Photo** (deferred): Counter, Number, Toggle, Single select, Multi select, Rating, Short text, Long text, Timer, Event log, Field-position picker, **Cycle path**, Computed, Section. *(confirmed 2026-08-17; Cycle path added 2026-08-20)*
  - **Timer** values are **editable after stop** (correct a late stop) and **nullable via an "unsure — no time" toggle** (submit no value instead of a wrong number).
  - **Event log** records **timestamped taps** — scouter-defined event buttons, each tap stored as `{type, t}` where `t` is seconds from a "match start" tap; enables counts and cycle-time analysis; taps are deletable before submit.
  - **Field-position picker** stores normalized `{x, y}` in 0–1 on the **season's uploaded game image** (one point or a list per entry). It carries **per-field alliance normalization**: the red alliance keeps raw coordinates; the blue alliance is mirrored — **horizontally, vertically, or both (configurable, with builder preview)** — so both alliances map to one canonical frame. This requires each entry to know its alliance. Shown in analytics as a heatmap/scatter over the game image.
  - **Cycle path** (Q3.3, added 2026-08-20). Captures a robot's cycle **routes** on the game image: the scouter taps an **ordered sequence of points** per cycle (e.g. pickup → score), and an entry holds a **list of cycles**. **Low fidelity by design** — a small **capped number of points per cycle** (target ≤ ~6) so the payload stays light; this is a rough sketch, not a precise trajectory. Uses the **same per-field alliance normalization** as the field-position picker (points stored as normalized `{x, y}` in 0–1, blue mirrored to the canonical frame). Rendered in analytics as **arrowed polylines** over the game image.
- **Form builder UI** (Q3.4). A **list-based builder + live-preview pane**; raw-JSON editing behind an "advanced" toggle. Design it cleanly. *(confirmed 2026-08-17)*
- **Version model** (Q3.5). A form has one **main/active version** plus **restorable secondary version snapshots**. Statistics always compute against the **main version**; a metric that references a field **absent from the main version** renders **"cannot calculate this metric"** until it is fixed/edited. Entries collected under other versions still aggregate through shared field `key`s. *(confirmed 2026-08-17)*
- **Form scope** (Q3.6). A **form belongs to a season**, not a competition. A season may hold several forms (match / super / …); every event in that season uses them. *(confirmed 2026-08-17)*
- **Delete semantics** (Q3.7). Deleting a form is a **cascade delete behind an explicit warning**, available to the **admin only**. *(confirmed 2026-08-17)*
- **Team-card flag** (Q3.8). Each field carries a **`show_on_team_card`** boolean so the strategy lead's quick team view is configurable per season rather than hard-coded. *(confirmed 2026-08-17)*
- **Conditional logic.** Simple rules only — `show this field if <field> <op> <value>`. No form-duplication feature; export/import of a form definition as JSON is retained. *(confirmed 2026-08-17)*
- **Every field is scoutable offline.** No per-field offline flag — offline is the normal operating mode, so all fields work offline. *(confirmed 2026-08-17)*

### 3.2 The three ways to build this

**Option A — One entries table with a JSONB payload (my recommendation).**

```
forms(id, season_id, kind, name, ...)
form_versions(id, form_id, version_no, published_at, is_locked)
form_fields(id, form_version_id, key, label, type, config, order, section, ...)
scouting_entries(
  id uuid,                -- generated on the client
  form_version_id,
  event_id, match_id, team_number, scouter_id,
  data jsonb,             -- { "auto_high_goals": 3, "climbed": true, ... }
  created_at, updated_at, client_updated_at, deleted_at
)
```

- Pros: no schema changes at runtime; one code path for everything; trivially syncable offline; trivial to add/remove/reorder fields; a single security policy covers all forms; Postgres indexes JSONB well (GIN, or expression indexes on hot fields).
- Cons: aggregate queries need casts (`(data->>'auto_high_goals')::numeric`); no database-level type enforcement — validation lives in the app and in a check function; slightly slower than native columns at very large scale.
- Mitigation for the cons: generate a **per-form-version SQL view** that flattens the JSONB into typed columns. Charts and statistics then query a normal-looking table. Optionally materialise it for speed.

**Option B — A real Postgres table per form (runtime DDL).**

- Pros: native types and indexes; the cleanest possible analytics SQL.
- Cons: the app needs privileges to run `CREATE TABLE` / `ALTER TABLE` in production, which with Supabase means handing the service-role key a lot of power — a real security concern; every new table needs its own row-level-security policies generated correctly or your data is exposed; deleting a field means either a destructive `DROP COLUMN` or dead columns forever; cross-form and cross-season queries need dynamic SQL; the offline client has to mirror an unknown, changing schema locally. This is the option that looks most natural and causes the most pain.

**Option C — EAV (entity–attribute–value): one row per answer.**

- Pros: typed value columns; fully flexible.
- Cons: pivoting back into rows-per-entry is painful and slow; row count explodes (60 fields × 6 robots × 100 matches ≈ 36,000 rows per event); offline sync at row granularity is fiddly.

**Recommendation: Option A**, with generated flattening views for analytics. It gives you Option B's query ergonomics without the operational risk.

**The MCP requirement (topic 20) strengthens this.** With Option A there is *one* stable table shape plus a machine-readable field dictionary, so an LLM discovers this year's game by reading one document. With Option B the model would have to discover N unknown tables with unknown column names every season, and any tool that queries them has to generate dynamic SQL — which is both fragile and the classic injection surface. Option A is meaningfully easier to expose safely to a model.

### 3.3 Proposed decisions

**Form versioning is immutable.** This is the thing people forget and regret. If you edit a form after scouting has started, previously collected entries must remain interpretable. So:

- Editing a *published* form creates **version N+1**; existing entries stay bound to version N.
- Field `key` values are permanent identifiers — you can rename the *label* freely, never the key.
- Deleting a field marks it deprecated in the new version; historical data is retained.
- Statistics that span versions must map fields explicitly (a field present in v1 but not v2 is either excluded or treated as null — we need a rule; see Q3.5).

**Field type catalogue (draft).** These are the field types I think an FRC form needs. Cut freely.

| Type | Notes |
|---|---|
| Counter | Big +/− buttons. The workhorse of match scouting on a phone. |
| Number | Free numeric with min/max/step. |
| Toggle / boolean | Did it climb, did it break down. |
| Single select | Radio / segmented control (e.g. climb level: none/low/high). |
| Multi select | Checkboxes (e.g. which scoring locations used). |
| Rating | 1–5 stars or slider, for subjective scouting. |
| Short text | Free text, indexed for search. |
| Long text / notes | Comments. |
| Timer / stopwatch | Accumulate time (e.g. time spent defending). **Editable after stop; nullable via an "unsure" toggle.** |
| Event log | Scouter-defined event buttons; each tap stored as `{type, t}` seconds from a "match start" tap. Enables cycle-time analysis. |
| Field position picker | Tap the season's game image; stores normalized `{x,y}` (0–1). **Per-field alliance normalization: red = raw, blue = mirrored H/V/both (configurable, with preview).** Shot-location heatmaps. |
| Cycle path | Tap an **ordered, low-fidelity list of points per cycle** (capped ≤ ~6; a list of cycles per entry) on the season game image; same alliance normalization as the position picker. Rendered as arrowed polylines. |
| ~~Photo~~ | **Deferred — not in v1.** |
| Computed | Read-only, derived from other fields by a formula. |
| Section / header | Layout only, no data. |

**Field configuration per field:** key, label, help text, type, required, default value, min/max, options list, section/page, display order, visibility condition, whether it appears in the quick summary, **`show_on_team_card`** (Q3.8), and — for the field-position picker — the **alliance-normalization axis** (none / horizontal / vertical / both). *(No per-field offline flag — **every field is scoutable offline**.)*

**Semantic metadata per field — added because of the MCP/LLM requirement (topic 20). [RAISED BY ME]**

An LLM cannot reason about a field called `f_17` holding the number `3`. For the AI stage to work at all, every field needs machine-readable meaning attached at the moment you create it in the form builder:

| Attribute | Example | Why the LLM needs it |
|---|---|---|
| `description` | "Game pieces scored in the upper goal during autonomous" | The only thing that tells the model what the number means. |
| `unit` | count / seconds / points / boolean / enum | Prevents nonsense comparisons and lets it phrase answers correctly. |
| `phase` | auto / teleop / endgame / post-match | "How is this team in auto?" requires knowing which fields are auto. |
| `direction` | higher_is_better / lower_is_better / neutral | Without it the model can't tell a good team from a bad one. "Fouls: 8" is bad; "Climbs: 8" is good. |
| `category` | scoring / defence / reliability / movement / driver skill | Enables "find me a defensive robot" without hard-coding this year's game. |
| `expected_range` | 0–9 | Lets the model recognise an implausible value instead of reporting it as fact. |
| `include_in_ai_context` | true / false | Lets you exclude noisy or private fields from anything sent to a model. |

This is a small addition to the form builder now and effectively impossible to backfill later — nobody will go back and describe 80 fields for three past seasons. It is also useful independent of AI: the same metadata drives chart axis labels, units in tables, default sort direction, and validation warnings. This is why I want it decided in topic 3, not topic 20.

**Scoring attributes (from topic 2).** Fields that contribute to the game score carry non-negative point value(s) — per enum option and per phase where relevant — captured here at creation time for the same non-backfillable reason. This is the **season scoring model** (§2.1, Q2.8 closed); success is derived as points > 0, not a separate predicate. Topic 9 implements the engine.

**Conditional logic:** simple rules only — `show this field if <field> <op> <value>`. Not a general expression language.

**Form portability:** export/import a form definition as JSON so it can be shared or version-controlled. *(No "duplicate last year's form" feature — dropped Q3.3 close.)*

### 3.4 Open questions

- ~~**Q3.1** — Do you accept **Option A (JSONB)**?~~ ✓ **CLOSED 2026-08-14: yes, Option A** (§3.1).
- ~~**Q3.2** — Do you accept **immutable form versioning**?~~ ✓ **CLOSED 2026-08-14: yes; label/range edits in place** (§3.1).
- ~~**Q3.3** — Which field types do you want, incl. field-position picking and event logs?~~ ✓ **CLOSED 2026-08-17: full catalogue except Photo (deferred); position picker uses the season game image with per-field alliance normalization; timer editable + nullable; event log = timestamped taps** (§3.1).
- ~~**Q3.4** — How is the form built in the UI?~~ ✓ **CLOSED 2026-08-17: list builder + live-preview pane; raw-JSON behind an advanced toggle** (§3.1).
- ~~**Q3.5** — What happens to a metric whose field only exists in some versions?~~ ✓ **CLOSED 2026-08-17: one main/active version + restorable snapshots; stats run on the main version; a metric on a missing field shows "cannot calculate this metric"** (§3.1).
- ~~**Q3.6** **[RAISED BY ME]** — Form↔competition relationship?~~ ✓ **CLOSED 2026-08-17: a form belongs to a season (not a competition); a season may hold several forms; all its events use them** (§3.1).
- ~~**Q3.7** **[RAISED BY ME]** — What does "delete a form" mean?~~ ✓ **CLOSED 2026-08-17: cascade delete behind an explicit warning, admin-only** (§3.1).
- ~~**Q3.8** **[RAISED BY ME]** — Per-field "show on team card" flag?~~ ✓ **CLOSED 2026-08-17: yes — `show_on_team_card` boolean per field** (§3.1).
- ~~**Q3.9** **[RAISED BY ME]** — Do you accept the **semantic metadata** block as part of every field, and which attributes are required?~~ ✓ **CLOSED 2026-08-14: yes; description/unit/phase/direction required, rest optional** (§3.1).
- ~~**Q3.10** **[RAISED BY ME]** — LLM-suggested field metadata from a pasted game manual?~~ ✓ **CLOSED 2026-08-17: wanted, deferred → §24 nice-to-have.**

---

## 4. Seasons, competitions & events

### 4.1 Confirmed requirements

- Data is organised by season and event name.
- **Forms belong to a season**, not an individual event; every event in the season uses the season's form(s) (Q3.6). *(amended 2026-08-17)*
- Search/filter data by event, and by all events within one season.
- **Active context (season + event).** The app has one app-wide active context. The **admin sets the default** (the `is_active` event and its season; season only if no event exists yet) and the app always **opens to it**. *(Q4.1, confirmed 2026-08-17)*
- A user may switch the season/event they are working in, but the switch is **session-only — not persisted**; reopening returns to the admin default. **Only the admin default is cached for offline use**; user overrides are never stored.
- The active context governs **both what the user browses and which event new scouting entries attach to**. Because a wrong context silently misattributes data, the switcher lives on a **dedicated context/landing page the user deliberately opens — never in the always-visible header/nav** (§16).
- **No event types — every event is a regular flat event** (Q4.2). No `type` field: district event, championship division, off-season and internal scrimmage are all just events (consistent with Q4.3, no division hierarchy). *(confirmed 2026-08-18)*
- **Events are weighted equally when aggregating across a season** (Q4.4). No event-level recency/weighting — a later event does not count more than an earlier one. *(confirmed 2026-08-18)*
- **No external data import in v1** (Q4.5). We do not import another team's data or scout from a stream, so there is **no data-import path and no `source` field on entries**. (Distinct from the deferred TBA/FRC schedule/result import — Q14.1, §24.) *(confirmed 2026-08-18)*
- **Data model** (Q4.2 close): `seasons(year, game_name, field_image_url)` → `events(id, season_id, name, code?, is_active, sort_order)`. `code` (the official event key, e.g. `2026isde1`) is **nullable and unused for now** — reserved only for the deferred TBA import (§24); `type`, `start_date`, `end_date` and `location` are **not modelled** (no import populates them). *(confirmed 2026-08-18)*
- **Aggregation scopes available everywhere:** single event, season (all events), multi-season / all-time, and a custom set of events. *(confirmed 2026-08-18)*
- **Events carry an admin-orderable `sort_order` within their season** *(added 2026-08-20 for Topic 9)*. Default = creation order; the admin can reorder a season's events, and all season-spanning stats/charts render competitions in this order (the full-season slope view, §9). **[RAISED BY ME]** — not in the original Q4.2 model; required by the year-view trend requirement. Reordering is display order only; it does **not** re-weight aggregates (Q4.4 unchanged).

### 4.2 Open questions

- ~~**Q4.1**~~ ✓ **CLOSED 2026-08-17:** global admin-set default context (season + event); users may override for the **session only** (not saved; only the default is cached offline); switcher on a dedicated page, not the header (§4.1, §16).
- ~~**Q4.2**~~ ✓ **CLOSED 2026-08-18:** no event types — every event is a regular flat event; `type` field dropped.
- ~~**Q4.3**~~ ✓ **CLOSED 2026-08-17:** flat event list — a championship division is just a regular event; no division hierarchy modelled.
- ~~**Q4.4**~~ ✓ **CLOSED 2026-08-18:** events weighted **equally** across a season; no event-level recency weighting.
- ~~**Q4.5**~~ ✓ **CLOSED 2026-08-18:** no external import in v1 — no data-import path, no `source` field on entries. `code` kept nullable; `start_date`/`end_date`/`location`/`type` dropped from the events model.

---

## 5. Users, roles, authentication & permissions

### 5.1 Confirmed requirements

- **Three global roles: Scouter, Lead, Admin** (Q5.1). Roles are **global**, not per-event; there is **no viewer/mentor role**. *(confirmed 2026-08-18)*
- **All data is visible to every role** (Q5.1, Q5.4). Scouters, leads and admins can view everything — statistics, dashboards, teams, forms and entries. No bias-hiding of aggregates from scouters. *(confirmed 2026-08-18)*
- **Form *templates* are admin-only; form *entries* are open to all** (Q5.1, reaffirms Q3.1/Q3.7). Creating, uploading/importing, editing and deleting a form *template* (definition/version) is **admin-only**. Submitting scouting *entries* against a form is open to **all users** — the normal scouting flow. *(confirmed 2026-08-18)*
- **Login: admin-provisioned username + password** (Q5.2). The admin creates each account and hands out its username and password. No self-service password reset in v1 — the admin re-issues. *(confirmed 2026-08-18)*
- **Only the admin creates users** (Q5.6); no self-registration.
- **Only the admin promotes/demotes** a user between Scouter / Lead / Admin (Q5.2).
- **Only the admin deletes users** (Q5.7). Deleting a user removes their access but **never deletes anything they created** — their scouting entries and any content remain, with authorship preserved. *(confirmed 2026-08-18)*
- **Scouters may edit their own entry for 5 minutes** after they create it; once the 5-minute window passes the row **locks** for them and only a Lead or Admin can change it (Q5.3). The window is measured from the entry's **client creation timestamp**, so it behaves **identically offline and online** — it is enforced on-device by the client clock, not at sync time. Scouters do **not** hard-delete entries — removal is a Lead/Admin action (soft-delete, §7.3). Leads and Admins may edit/manage any entry at any time. *(confirmed 2026-08-18)*
- **Offline session persistence: ~30 days, refreshed on use** (Q5.2). A scouter authenticates once (e.g. on Wi-Fi at the hotel) and stays logged in offline through the whole event. *(decided-by-Claude 2026-08-18)*
- **Switch-scouter quick action on shared devices** (Q5.8). Shared team tablets get a fast "switch scouter" action instead of full logout/login, and **every entry records the scouter who actually entered it**. *(confirmed 2026-08-18)*
- **No user audit log** (Q5.5). The app keeps no who-did-what-when log of user actions. *(confirmed 2026-08-18)* *(Entry edit-history was resolved on Topic 13 close: **no full per-entry history** either — routine edits overwrite last-write-wins; only §7.3 divergence preserves the superseded copy.)*
- **Authorization is enforced in the server use-case layer and surfaced in the UI — not in the database** (Q5.1 close). **No Postgres Row-Level Security, no per-row policies.** Because all traffic already goes through the server API (Q15.1, the single control point), each use case checks the caller's role and the UI hides or disables actions a role may not perform. *(confirmed 2026-08-18)*

### 5.2 Role & permission matrix (confirmed)

| Capability | Scouter | Lead | Admin |
|---|:---:|:---:|:---:|
| Log in; view all data (stats, dashboards, teams, forms, entries) | ✓ | ✓ | ✓ |
| Submit scouting entries; edit **own** entries (≤ 5 min, then locked) | ✓ | ✓ | ✓ |
| Manage **all** entries (edit / fix / reassign / soft-delete others') | — | ✓ | ✓ |
| Add a team to the **do-not-pick** list (reason required); may not edit or remove one (§12.2) | — | ✓ | ✓ |
| Create a **draft statistics page** on the active competition (session-only, not saved, discarded on exit) | — | ✓ | ✓ |
| Create / edit / **save** statistics & dashboards (persistent) | — | — | ✓ |
| Create / upload / edit / delete **form templates** | — | — | ✓ |
| Manage events & the active-context default (§4.1) | — | — | ✓ |
| Build / reorder **pick lists**; edit or remove do-not-pick entries; record the **alliance bracket** (§12.2) | — | — | ✓ |
| Create / delete users; promote / demote roles | — | — | ✓ |

The **draft statistics page** (Lead) is an ephemeral dashboard scoped to the current active competition: built during a session, viewable live, **never persisted to the database**, and discarded on **exit — defined as logout or closing the app/tab**. It is held in session-scoped memory, so it **survives navigation between pages within the session** but is gone once the lead logs out or the app closes (and can be dismissed manually). Persistent/saved statistics are admin-only. See Topic 10 (§10.2) for the dashboard mechanics; this fixes only who may create ephemeral vs. saved ones.

### 5.3 Resolved questions

- ~~**Q5.1**~~ ✓ **CLOSED 2026-08-18:** three global roles (scouter/lead/admin), no viewer/mentor, not per-event; matrix §5.2; form templates admin-only, entries open to all.
- ~~**Q5.2**~~ ✓ **CLOSED 2026-08-18:** admin-provisioned username + password; admin promotes/demotes; ~30-day offline session.
- ~~**Q5.3**~~ ✓ **CLOSED 2026-08-18:** scouters edit own entries for **5 min** from creation (by client timestamp, offline & online), then the row locks; leads/admins anytime; scouters don't hard-delete.
- ~~**Q5.4**~~ ✓ **CLOSED 2026-08-18:** all data visible to all roles; no bias-hiding.
- ~~**Q5.5**~~ ✓ **CLOSED 2026-08-18:** no user audit log (entry edit-history resolved on Topic 13 close — no full per-entry history either).
- ~~**Q5.6**~~ ✓ **CLOSED 2026-08-18:** admin-only user creation; no self-registration.
- ~~**Q5.7**~~ ✓ **CLOSED 2026-08-18:** admin-only user deletion; deletion preserves everything the user created.
- ~~**Q5.8**~~ ✓ **CLOSED 2026-08-18:** switch-scouter quick action on shared devices; entries record the actual scouter.

**Still open (dependency, not blocking Topic 5):** Q20.6 — whether to reserve a read-only `service`/`agent` role for a future non-human (MCP) caller. App-layer authorization means it can be added later with no rework. **[RAISED BY ME]**

---

## 6. Scouting data entry (runtime UX)

### 6.1 Why this matters more than it sounds **[RAISED BY ME]**

This is where 95% of the app's actual usage happens: a student standing in a loud arena, phone in one hand, watching a robot move fast, with 2 minutes and 30 seconds of match plus ~10 seconds between matches. If this screen is even slightly awkward, the data quality collapses and every graph downstream is worthless. I'd like to spend real time on this topic.

### 6.2 Confirmed requirements

- **Manual match selection in v1** (Q6.1). No assignment system in v1: the scouter **manually picks the match number and the team/robot** to scout. The schedule-driven assignment flow (auto-fill *"Match 34 — you are watching Red 2, team 1577"*) is **deferred to §24 nice-to-have**, gated on TBA/schedule import (§14). *(confirmed 2026-08-18)*
- **Station-based assignments when built** (Q6.2). When the assignment system lands it is **station-based / fixed positions** — the scouter at a fixed station always watches the same alliance slot (station 1 → Red 1). Recorded now so the deferred design is fixed. *(confirmed 2026-08-18)*
- **Collapsible phase sections** (Q6.3). The entry form is organised into phase sections (Autonomous → Teleop → Endgame → Post-match) that the scouter can **open, close, and return to at will** — not one-way paging. A scouter can reopen an earlier phase to fix a value. *(confirmed 2026-08-18)*
- **Sticky match timer** (Q6.4). A full-match timer is **pinned to the top of the screen and stays visible while scrolling**. Its **phases (auto / teleop / endgame) and their countdown durations are part of the form definition**, reusing the same phase vocabulary as field metadata (§3.3) and the scoring model (§2.2). A **manual "Start match" button** starts it (we cannot hook the field). The timer is **display-only guidance**: it does **not** open or close phase sections, does **not** lock or gate any field, and does **not** auto-submit — scouters open phases themselves. *(confirmed 2026-08-18)*
- **Mandatory robot status** (Q6.5). Every match entry carries a required top-level **`robot_status` ∈ {played, no-show, disabled, broke-down-at-T}**, chosen before the scoring fields. For **no-show** and **disabled** the scoring fields are **hidden — no zeros are ever entered**; the entry records only the status. `broke-down-at-T` also captures the breakdown time. *(confirmed 2026-08-18)*
  - **Statistics treatment.** Performance metrics (scores, piece counts, averages) **count `played` and `broke-down-at-T`** — a robot that played then died genuinely underperformed, so its partial data is real observed performance — and **exclude `no-show` and `disabled`** entirely (never counted as zero). A separate **reliability / availability metric** counts `played` + `broke-down-at-T` as *available* and `no-show` + `disabled` as *missed*, and flags breakdowns. Topic 9 implements this metric (§9.2). *(confirmed 2026-08-18)*
- **Both orientations, phones and tablets** (Q6.6). The data-entry UI works in **both portrait and landscape** and adapts its layout — a wider (landscape) screen, typical on a tablet/iPad held sideways, places fields side-by-side; a phone stacks them. Usable on **phones and tablets/iPads**. *(confirmed 2026-08-18)*
- **Arena-comfort features** (Q6.7). Ship all of: **large touch targets, an outdoor-readable high-contrast mode, a large-text option, haptic feedback on taps, screen-wake-lock (no dimming mid-match), thumb-reachable primary actions, and counters instead of the keyboard** wherever possible. *(confirmed 2026-08-18)*
- **Practice / training mode** (Q6.8). A mode to enter data against a **real form** without polluting the database: practice entries are marked as such and are **never stored or synced** — discarded on exit. *(confirmed 2026-08-18)*
- **Undo on all repeatable inputs.** Undo is available on **counters, event-log taps, field-position-picker points (undo last point / clear all), multi-select, and timer reset** — anywhere a stray or mistaken tap needs stepping back. *(confirmed 2026-08-18)*
- **Explicit submit.** Submitting is a deliberate action showing a **confirmation summary of the whole entry** before it commits; nothing reaches the shared data on a stray tap. In v1 (no assignments) submit returns to a fresh manual selection. *(confirmed 2026-08-18)*
- **Never lose data.** Every interaction writes a **local draft immediately** (IndexedDB, §7.3); the draft survives a browser crash, a dead battery, or an accidental back-swipe and is **recoverable**; an explicit submit promotes the draft into the sync outbox. *(confirmed 2026-08-18)*

### 6.3 Resolved questions

- ~~**Q6.1**~~ ✓ **CLOSED 2026-08-18:** manual match + team selection in v1; schedule-driven assignment system → §24 nice-to-have, gated on TBA import.
- ~~**Q6.2**~~ ✓ **CLOSED 2026-08-18:** when built, assignments are **station-based / fixed positions** (station 1 → Red 1).
- ~~**Q6.3**~~ ✓ **CLOSED 2026-08-18:** **collapsible phase sections**, openable/closable and reopenable — not one-way paging.
- ~~**Q6.4**~~ ✓ **CLOSED 2026-08-18:** **sticky top match timer**; phases + countdown durations are part of the form definition; manual "Start match"; **display-only** — never gates fields, opens phases, or auto-submits.
- ~~**Q6.5**~~ ✓ **CLOSED 2026-08-18:** mandatory top-level `robot_status` (played / no-show / disabled / broke-down-at-T); no-show & disabled hide the scoring fields (no zeros). Stats: performance metrics count `played` + `broke-down`, exclude `no-show` + `disabled`; a separate reliability/availability metric reads the status (Topic 9).
- ~~**Q6.6**~~ ✓ **CLOSED 2026-08-18:** both portrait and landscape; adaptive layout; phones and tablets/iPads.
- ~~**Q6.7**~~ ✓ **CLOSED 2026-08-18:** all arena-comfort features (large targets, high-contrast/outdoor mode, large text, haptics, screen-wake-lock, thumb-reachable actions, counters over keyboard).
- ~~**Q6.8**~~ ✓ **CLOSED 2026-08-18:** practice/training mode — real form, entries never stored/synced.

---

## 7. Offline-first & synchronisation

### 7.1 Confirmed requirements

- Works fully with no internet: loads existing data, allows changes.
- When internet returns, changes upload.
- **Statistics and rankings compute on-device while offline** (Q7.4). The metric engine runs in the browser from the shared `packages/shared` engine — the *same code* as server-side, so results cannot diverge. Metric calculations are expected to stay simple. *(confirmed 2026-08-14)*

### 7.2 The thing you must know about FRC venues **[RAISED BY ME]**

At official FRC events, **wireless communication in the pits and stands is heavily restricted** — the field's radio spectrum is protected, event WiFi is often unavailable or unusable, and cellular is frequently saturated by thousands of people in a metal arena. In practice, "offline" is not the exception at a competition; **it is the normal operating mode**. This inverts a normal app's design: the local database is the source of truth during an event, and the cloud is where it eventually lands.

Consequences we should decide on:

1. The app must be **installable as a PWA** and fully functional with the network switched off, including cold start (app shell cached, no network request required to boot).
2. Sync happens opportunistically whenever a connection appears, and there must be a **visible, trustworthy sync status** — how many entries are pending, when the last successful sync was, and a manual "sync now" button.
3. We need a **fallback transfer path** for when there is genuinely no internet all weekend. Realistic options: a phone hotspot used by one device at a time outside the pit area; a local WiFi network with a laptop acting as a hub; **QR code chains** (scouter's phone renders a QR, a central tablet scans it) — this is a proven FRC pattern and requires no network at all; or plain file export/import over USB or AirDrop.

### 7.3 Confirmed decisions

- **Local store:** IndexedDB (via Dexie or similar), scoped to the **active (admin-default) competition only** — its form definitions and versions, the team list, that event's raw entries (the source the on-device engine computes statistics from — no pre-computed values stored), the outbox, and **cached user records so scouters can log in and switch user offline** on a shared device. No multi-season/multi-event bulk caching on-device.
- **Client-generated UUIDs** for all records, so an offline device can create records that will never collide on upload.
- **Outbox / operation log** pattern: every local mutation is appended as an operation with a client timestamp and a monotonic counter; sync replays operations to the server; the server acknowledges and the client prunes.
- **Conflict policy — last-write-wins live; only genuine divergence goes to review** (optimistic concurrency). Every row carries a server-assigned **version** (revision counter). The **edit function is separate from sync**: each mutation records **the base version it started from**. On sync the server compares that base to the row's current version:
  - **Fast-forward** — base matches the DB, or the UUID is new: apply and bump the version, **last-write-wins, no review.** Covers every normal case — a scouter's own 5-min self-edit, a re-sync, a re-scanned QR batch.
  - **Divergence** — base is stale (two edits branched from the same ancestor on different devices): a **genuine conflict** → keep the **latest as the live value** so the app stays usable, **preserve the superseded version**, and flag the row for review. No automatic field-level merge. **The "review conflicts" screen requires a human decision from a lead *or* an admin** (both roles can resolve; a scouter cannot) — a flagged conflict is not considered settled until one of them acts.

  For form definitions, offline admin edits are **not allowed** — form changes require connectivity (avoiding two divergent form schemas). *(Q7.5, refined 2026-08-19.)*
- **Editable offline:** **scouting entries** (create + 5-min self-edit) and the **alliance-selection surfaces** — the admin-only **pick lists** and **alliance bracket**, plus a **lead's do-not-pick addition** — all syncing through the same outbox. *(Extended on Topic 12 close, §12.2.)* A pick-list **reorder** is a single operation stamped with the list's own version, not a per-row write (§12.2).
- **Statistics offline:** the **main statistics page works offline for viewing** — rankings and metrics are computed on-device from the raw entries — but **cannot be edited offline**. A lead's **draft statistics page can be created offline** and, as always, is **discarded on exit** (§5.2). **Saving** a persistent admin dashboard needs connectivity; a saved dashboard is viewable from cache offline but is built/edited only online. Offline stats naturally reflect only the entries the device currently holds — they become complete once it has gathered the others (sync or QR).
- **Not editable offline:** form definitions and user management.
- **Offline dataset — the full active competition, and it fits easily.** Offline analysis needs the real source data on the device: **all active-competition forms/versions, the user list, and every raw scouting entry** (the "statistics details"). **Nothing is pre-computed** — the on-device shared engine calculates each metric from the raw entries (matches §17.1: store raw, aggregate on the client). Feasibility is not a concern: with **no photos** (position fields store only normalized x,y coordinates), one event is a **few MB of JSON** (~600 match entries × ~1–2 KB plus super-scouting) — far within IndexedDB's limits, so it loads and indexes fast. The cache is bounded to the **active competition only** — no multi-season/multi-event bulk is held on-device.
- **Lead-approved local wipe.** A "clear this device's offline data" action gated behind a **constant code held by leads** — a safety valve to reset a misbehaving or handed-down device. The wipe is **refused for any record without a cloud ack** (durability rule below), so it can never destroy unsynced data.
- **Soft deletes** everywhere (`deleted_at`), because a delete that happened offline needs to propagate and win predictably.
- **Durability rule — never prune before a cloud ack:** a device discards a local record from its outbox **only** after receiving a server acknowledgment for that exact UUID. Nothing else — closing the app, a device handoff, a QR transfer, or the lead-approved wipe — ever removes an unacked record. This is the single load-bearing guarantee against data loss.
- **Fallback transfer = QR, in v1** (Q7.1): the sender batches all pending outbox entries, **compresses** them, and renders an **animated multi-frame QR** (fountain-coded, order-independent); a central tablet scans the whole batch in one continuous scan — not one code per entry. Matches the real workflow (Q7.2): a **central tablet collects scouters' data by QR, then a runner carries it outside the arena every few games to sync to the cloud.**
- **QR transfer is additive & idempotent:** scanning a device's animated-QR dump **copies** its records to the receiver with their **original UUIDs, scouter, and client timestamps** (not re-authored). The sender keeps its outbox pending — a QR scan is a backup hop, not a confirmed sync — so the data now exists on both devices and is strictly more durable. When either device (or both) later reaches the internet, each uploads independently; because the server keys on UUID, a second upload of the same record is an **idempotent upsert** resolved by the base-version rule above (no-op if identical, fast-forward if linear, flagged for review only on genuine divergence), never a duplicate. No device need be designated "the syncer," and re-scanning the same batch is harmless.

### 7.4 Resolved questions

- ~~**Q7.1** — Is QR-code transfer in scope for v1 or later?~~ ✓ **CLOSED 2026-08-19: in v1, as an animated + compressed multi-frame QR** (§7.3).
- ~~**Q7.2** — Any realistic connectivity at your events?~~ ✓ **CLOSED 2026-08-19: yes, outside the arena** — one tablet is walked out to sync every few games; QR feeds that tablet inside (§7.3).
- ~~**Q7.3** — Which entities must be editable offline?~~ ✓ **CLOSED 2026-08-19: scouting entries + pick list (admin-only)** editable offline; form defs / user mgmt / persisting dashboards not — but a lead's ephemeral draft stats page does work offline (§7.3). *(Topic 12 close extends the offline-editable set to the do-not-pick list and the alliance bracket, and allows a lead's do-not-pick addition offline — §12.2.)*
- ~~**Q7.4** — Should statistics and charts be computed on-device while offline?~~ ✓ **CLOSED 2026-08-14: yes — shared TS metric engine runs in the browser** (§7.1).
- ~~**Q7.5** — When two devices edited the same entry, who wins?~~ ✓ **CLOSED 2026-08-19: last-write-wins live via base-version check; only genuine divergence is flagged for lead/admin review, superseded version preserved** (§7.3).
- ~~**Q7.6** **[RAISED BY ME]** — Device-to-device sync on a local network?~~ → **§24 nice-to-have** (local-network hub, no internet; not USB).
- ~~**Q7.7** **[RAISED BY ME]** — How much local storage; photos?~~ ✓ **CLOSED 2026-08-19: cache the full active competition (a few MB — no photos, x,y coords only) so offline stats work; active-competition-only bound; lead-code local wipe** (§7.3).

---

## 8. Realtime & live updates

### 8.1 Confirmed requirements

- **No realtime / live push in v1 — periodic refresh instead** (Q8.1). Cross-device live updates (Supabase Realtime / WebSockets) are **nice-to-have** (§24). In v1 a device sees other devices' changes through the normal refresh triggers — re-querying on screen entry, pull-to-refresh, manual refresh, re-sync on reconnect — **plus a background auto-refresh every 45 seconds** on data-bearing screens while online. *(confirmed 2026-08-19)*
- **Optimistic local UI stays — core, not deferred.** A user's own action appears instantly on their own device with no manual refresh (except while offline). This is a *local* mechanism — the same one offline writes use (§7) — not cross-device push, so it is not part of the deferral. *(confirmed 2026-08-14)*
- **Connection-state indicator**, always visible, three states: **online / syncing / offline** (§7.2, §16). *(confirmed)*
- Needing a full app restart to see changes is a caching bug to design out (PWA: cache the shell, network-first for data). *(confirmed)*

### 8.2 Resolved questions

- ~~**Q8.1** — What genuinely needs to be live?~~ ✓ **CLOSED 2026-08-19: nothing is push-live in v1; a 45-second auto-refresh (plus the refresh triggers) covers cross-device propagation. Supabase Realtime → §24.**
- ~~**Q8.2** — Live "match in progress" view?~~ → **§24 nice-to-have** (requires connectivity often absent at a venue).
- ~~**Q8.3** **[RAISED BY ME]** — Presence (who's online / who hasn't submitted)?~~ → **§24 nice-to-have.**
- ~~**Q8.4** **[RAISED BY ME]** — Notifications?~~ → **§24 nice-to-have** (in-app vs. push decided then).

> **Dependency note — RESOLVED 2026-08-31 (Topic 12 close):** §12's **live cross-off** is instant on the **admin's own device** (optimistic local write), reaches **online viewers on the 45-second auto-refresh** or a manual refresh, and offline viewers only on sync. Because the pick list is **single-editor (admin-only)**, **the admin's device is the declared source of truth during alliance selection** and every other screen is advisory (§12.2).

---

## 9. Statistics engine & computed metrics

### 9.1 Confirmed requirements

- Statistics per game/form, viewable per competition or across all competitions of a year.
- Statistics feed the graph/visualisation pages.

**Confirmed at Topic 9 close (2026-08-20):**

- **Menu builder, no free-text formula language** (Q9.2). Metrics are built by choosing field(s) → aggregation → filters, with multi-field arithmetic (at least summing, e.g. `auto + teleop`) but no typed expression evaluator. When creating a statistic page you can also set a **display limit/target value** (the game's cap for that metric) to draw as a reference on the chart/table.
- **Match-subset selection, no recency weighting** (Q9.6). A metric may compute over **last N matches** and/or with **specific matches manually excluded** (or both); every included match counts equally (consistent with Q4.4).
- **Standard deviation is displayed, not a ranking order** (Q9.5). Consistency is available and shown next to averages, but the app does not sort/rank teams by it. **Reliability is a first-class ranking input** (robot-status handling, §9.2).
- **No cross-event percentile / z-score normalisation** (Q9.7). Percentile stays available as an ordinary aggregation over a team's own entries.
- **Minimum sample size = 1** (Q9.8). The engine only guards against empty/zero-division; it does not flag or refuse low-sample teams.
- **One canonical entry per (team, match)** (Q9.4). The normal path is edit/review of a single entry — no deliberate multi-scout redundancy in v1. The only collision is a sync conflict, resolved **last-write-wins live, superseded version preserved and flagged for lead/admin review** — identical to §7.3 (Q7.5). Statistics read the live value, so averages never double-count.

**Statistics pages (confirmed 2026-08-20; page/chart mechanics live in Topic 10):**

- **Team stat page** — reachable from a team's page; shows *that team's* stats over the **matches of the active competition** only. Fixed, always available.
- **General stat-page builder** — a separate route for pages spanning **many teams and/or a full season**. Per §5.2: **admins create *and save*** persistent pages; **leads create/explore but cannot save** (session-only draft pages, discarded on exit).
- **Full-season ordering** — competitions render in **creation order by default** so a metric's **slope from first to last competition** (and per-match-per-competition across the year) reads left-to-right; the admin can **reorder** competitions within a season. Backed by a persisted `sort_order` on events — see §4.1 amendment.
- Heatmap-per-match and cycle/event-log views raised in Q9.1 are **Topic 10 visualisation concerns**, driven by these same metrics.

### 9.2 Proposed decisions

**Two layers of statistics.**

*Layer 1 — Metric definitions (created in the app, per form):* a named, reusable calculation over form fields.

- Field combination via a **menu builder, not a typed formula language** (Q9.2): choose field(s) → aggregation → filters. Multi-field arithmetic is supported at least for summing (e.g. `total_pieces = auto_pieces + teleop_pieces`); there is **no free-text expression evaluator**.
- Aggregation over a team's entries: average, median, sum, min, max, count, standard deviation, percentile, rate (`count where condition / total`), trend/slope over match number, "last N matches" average, best/worst.
- Filters: exclude no-shows and disabled robots; only qualification matches; only a chosen date range or event set; **a match subset — last N matches and/or specific matches manually excluded** (Q9.6, no recency weighting).
- Metrics are versioned alongside forms and are reusable across every chart, table and ranking.

*Layer 2 — Derived FRC statistics from official results:* OPR, DPR, CCWM, win rate, average ranking points, schedule strength. These come from match scores, not from your scouting, and are a valuable cross-check. **Deferred → §24 nice-to-have (Q9.3), gated on TBA import (§14):** not built until official results are imported. (confirmed 2026-08-20)

**Robot-status handling (confirmed at Topic 6 close, Q6.5).** Every match entry carries a top-level `robot_status` ∈ {played, no-show, disabled, broke-down-at-T}. The engine applies a fixed rule: **performance metrics include `played` and `broke-down-at-T`** (partial data from a robot that died is real observed performance) and **exclude `no-show` and `disabled`** (they must never enter an average as zero). A first-class **reliability / availability metric** reads the same field — `played` + `broke-down-at-T` count as *available*, `no-show` + `disabled` as *missed*, with breakdowns flagged. This is why the status is a fixed skeleton field, not a form field.

**Reliability is surfaced and used, not just available (confirmed 2026-08-20).** The engine exposes explicit counts per team — **breakdowns** (`broke-down-at-T`), **no-shows** (`no-show`), **disabled** (`disabled`) — plus a derived **availability rate** = available / (available + missed). These counts are shown alongside performance metrics in the statistics, ranking and pick-list views, and reliability is a **first-class input to alliance selection**: a high scorer that frequently breaks down or no-shows must not out-rank a slightly lower but dependable robot. Standard deviation/consistency is displayed but never used to order rankings (Q9.5); reliability, by contrast, *is* a ranking input.

**Aggregation scopes (revised 2026-08-20, Q9-close):** **per team per competition**, **per team per match** (a team's per-match series within a competition), **per scouter per competition** (reliability), and **per team per season** (needed for the full-season/slope view). *(Per-team all-time, general per-match-across-robots, and per-scouter-per-match dropped.)*

**Where computation happens:** SQL views/functions in Postgres for the online case (fast, close to the data), and a shared TypeScript implementation for the offline case (required — Q7.4 closed: offline stats run in the browser from the shared engine). Both must be driven from the same metric definition and tested against each other, otherwise they will silently disagree.

### 9.3 Open questions

- ~~**Q9.1** — What statistics do you *actually* use?~~ ✓ **CLOSED 2026-08-20:** all listed Layer-1 aggregations are first-class; everyday use is averages (game pieces) and success rates (e.g. climb). Heatmap-per-match and cycle/event-log views → Topic 10 (visualisation), same metrics.
- ~~**Q9.2** — Formula language or menu builder?~~ ✓ **CLOSED 2026-08-20:** menu builder, **no free-text formula language**; supports multi-field arithmetic (summing, e.g. `auto + teleop`) and a per-page display limit/target value (§9.1).
- ~~**Q9.3** — OPR/DPR/CCWM/EPA from official results?~~ ✓ **CLOSED 2026-08-20:** wanted → §24 nice-to-have (Layer 2), gated on TBA import (§14).
- ~~**Q9.4** **[RAISED BY ME]** — Duplicate (team, match) resolution?~~ ✓ **CLOSED 2026-08-20:** one canonical entry per (team, match) via edit/review — no deliberate redundancy; sync collisions resolved last-write-wins per §7.3 (Q7.5). No averaging of duplicates.
- ~~**Q9.5** **[RAISED BY ME]** — Consistency first-class in ranking?~~ ✓ **CLOSED 2026-08-20:** standard deviation is a **displayed** metric but **not** a ranking sort key; reliability (robot-status) *is* a first-class ranking input.
- ~~**Q9.6** — Recency weighting?~~ ✓ **CLOSED 2026-08-20:** **no** weighting; instead a selectable **match subset** — last N and/or manually excluded matches (§9.1/§9.2 filters).
- ~~**Q9.7** **[RAISED BY ME]** — Percentile / z-score cross-event normalisation?~~ ✓ **CLOSED 2026-08-20: no.** Percentile stays available as a plain aggregation.
- ~~**Q9.8** **[RAISED BY ME]** — Minimum sample size flag?~~ ✓ **CLOSED 2026-08-20:** minimum = **1** (guard against empty/zero-division only); no low-sample flag or refusal.

---

## 10. Dynamic dashboards, graphs & visualisations

**Status: CLOSED 2026-08-21 (Q10.1–Q10.10).** Confirmed requirements below; §10.2 holds the design detail; §10.3 records the closed questions.

### 10.1 Confirmed requirements

- Create statistics and graph pages **dynamically from inside the app**; point charts at specific form fields/metrics.
- **View scope: one event or the whole season (or a season subset).** Metric definitions are season-level and event-agnostic; **scope binds at the dashboard level** (§10.2).
- **Chart set for v1 (Q10.1): everything on the list, high-analysis, including pie/donut.** Bar (grouped/stacked), horizontal bar, line/progression, scatter, radar/spider, box plot, histogram, heatmap (field-position overlay and team×metric), pie/donut, data table with sorting + inline sparklines, single-number stat cards, **cycle-path overlay**, plus the added high-level extras in §10.2 (stacked area, bump/rank-over-time, bullet/target gauge, small-multiples).
- **Dashboards are shared team assets (Q10.2).** Any **saved** dashboard is visible to **all users**; **drafts are private to their session and never shared**. Saving is admin-only; leads build session-only drafts; scouters view. Draft-vs-saved and offline behaviour are unchanged from Topic 5 §5.2 / Topic 7 §7.3 (saving a dashboard needs connectivity; a lead's draft stats page works offline).
- **Built-in dashboards operate on the active event, or a selected event if changed** (season-wide where the view is a season view). The `scope.events` field stays in the config schema (per-event or per-season).
- **Charting library (Q10.6): Recharts** for all standard charts, with the **image overlays (field-position heatmap, cycle-path) and the team×metric heatmap-table hand-built in SVG/Canvas.** ECharts rejected as overkill for the dataset size; design quality is a first-class goal.
- **Mobile density (Q10.5):** when a chart's dimension is **all teams in the event/season**, the phone view shows **top-N and bottom-N** (default **top 8 + bottom 8**) with a "show all" expansion; when the view is **per-game / per-team / any non-all-teams slice, show everything**. **Per-team match views are designed for up to 14 matches** per team at an event.
- **View-time metric selector + expand-to-stack (Q10.9/Q10.10):** a chart can add or swap a metric that shares its **X dimension** (comparable axis/unit); "expand" stacks instances vertically, **capped at 4**.
- **Next-year form change (Q10.3):** a dashboard whose fields no longer exist is **read-only** (renders what it can, flags missing fields). A **re-map wizard** to the new form is **nice-to-have (§24)**.
- **Configurable operational (meta) statistics (Q10.9-ops):** the general/operational statistics page is **built with the same builder, not static**, pointed at **app-metadata data sources** instead of form fields (§10.2). Meta-field catalogue **finalized on Topic 13 close (2026-08-21)** — see §10.2.
- **Deferred to §24 nice-to-have:** export (PNG/CSV/Excel/PDF, Q10.4); drill-down click-through (Q10.7); scouted-vs-official comparison (Q10.8, gated on TBA import).

### 10.2 Design detail

**A chart is a saved configuration**, stored in the database, referencing a form version plus fields/metrics:

```
{
  type: "bar",
  scope: { season: 2026, events: ["2026isde1"], match_types: ["qual"] },
  dimension: { field: "team_number" },          // x axis / rows
  series: [ { metric: "avg_total_pieces" },      // y axis
            { metric: "avg_auto_pieces" } ],
  filters: [ { field: "robot_status", op: "=", value: "played" } ],
  sort: { by: "avg_total_pieces", dir: "desc" },
  limit: 20,
  options: { stacked: true, show_error_bars: true }
}
```

**Metric definitions are season-level and event-agnostic.** A metric/statistic definition never names an event — it is defined for the whole season. **Scope is bound at the dashboard level, not the metric level**, and is fixed when the dashboard is created to either (a) the **full season or a subset of it**, or (b) **exactly one event**. This applies identically to **draft** (lead) and **saved** (admin) dashboards. Some metrics are only meaningful on the current event in practice, but that is a property of the view's scope, not of the definition.

**Chart types (confirmed, Q10.1 — the full high-analysis set):** bar (grouped/stacked), horizontal bar, line (over match number — the progression chart), scatter (two metrics against each other, for finding complementary partners), radar/spider (multi-metric team comparison — very popular in FRC), box plot (distribution and consistency), histogram, heatmap (field position data, or team × metric), pie/donut, **data table** with sorting, inline sparklines and **value shading** (see below) — rendered as a **fixed-height scroll viewport (~8 rows visible) with a frozen header row and a sticky label/first column**, so a table of many scouters or teams stays readable without pushing the rest of the page down — single-number "stat cards", and **cycle-path overlay** (arrowed polylines of Cycle-path routes drawn over the game image, aggregable across matches/teams). **Added high-level extras:** **stacked area** (share of contribution over matches), **bump / rank-over-time** (how a team's ranking moves across the event), **bullet / target gauge** (value against the per-page display limit/target from §9.1), and **small-multiples** (the same chart repeated per team in a grid — the static sibling of expand-to-stack).

**Metrics are type-agnostic.** A metric works on **any field type** — counts, numbers, booleans, ratings, single/multi-select categories, and **time/durations** (Timer values and Event-log-derived cycle times) — with the aggregation menu offering the operations valid for that type (e.g. avg/median/min/max/stddev for numeric and time, rate/percentage for boolean, mode/distribution for categorical). Units and default sort come from the field's semantic metadata (§3.3).

**Dashboards** are ordered collections of charts on a responsive grid, saved with a name, scoped as above (season/subset or single event), and shareable by link. A dashboard has global filters (event, match type, team subset) that all its charts inherit unless overridden.

**Tile layout on the grid.** Charts are placed on a **12-column responsive grid**. Each tile picks a **width span** — e.g. a stat card = 3 columns (four per row), a standard chart = 6 (two per row), a wide chart/table = 12 — and tiles are **drag-reordered**. This is how you put, say, two stat cards on one row. On narrower screens the grid **reflows** to fewer columns and finally to a single stacked column (phone), preserving order. Keep the interaction simple: choose a size (small / medium / full) and drag; no free-pixel positioning.

**View-time metric selector + expand-to-stack.** A chart can expose an **interactive metric selector** (a dropdown shown while viewing, not only at build time) that adds or swaps a metric **sharing the chart's X dimension** (usually match/game) with a comparable axis/unit — e.g. a per-match progression chart on a team page swapping between teleop GP, auto GP, etc. An **"expand"** action stacks instances of the chart **vertically, capped at 4**, each showing a different metric from that pool, so multiple metrics are visible at once one below the other. This is a **view interaction**, not additional saved charts.

**Saved vs. draft dashboards (role split — see Topic 5 §5.2).** Creating and **saving** a persistent dashboard is **admin-only**. A **Lead** can build a **draft statistics page** using the same builder, scoped like any dashboard (season/subset or one event) — though **offline it can only compute over the cached active competition** (§7.3) — but it is **held in session memory only**: viewable live, **never written to the database**, and discarded on **exit** — defined as **logout or closing the app/tab**. It therefore **survives navigation between pages within the session** (the lead can leave it and come back) and can be dismissed manually, but nothing persists once the session ends. **Scouters** view dashboards but create neither kind.

**Builder UX:** pick a chart type → pick the dimension → add one or more series (field or metric + aggregation) → add filters → live preview → save. Aim for "understandable by a 15-year-old in 5 minutes", not Tableau.

**Standard pages that should exist out of the box** (rather than requiring everyone to build them from scratch):

- Team page: all metrics for one team, match-by-match table, progression charts, notes. *(No photos in v1 — Photo field deferred, §3.1.)*
- Ranking page: metrics table (admin-built, one per season) with column-sort and a weighted-composite mode. *(Full UX: §11.2.)*
- Compare page: 2–6 teams — head-to-head "final score" for 2, radar + table for 3–6. *(Full UX: §11.2.)*
- Match preview: six user-entered teams by alliance, with a summed-average score prediction and reliability flags. *(Full UX: §11.2.)*
- Coverage/quality page: which matches are missing data. *(→ §24, gated on TBA schedule import, per Topic 13.)*

Each of these is a **built-in dashboard**. It is reachable **in-context from its home page** — the **team dashboard** from a team page, the **ranking dashboard** (event-scoped) from the ranking page, the **compare dashboard** from the compare page — where the required input (team number, event, team set) is supplied automatically by that page.

**General Dashboards page (the hub).** A dedicated page lists **every dashboard for the active season**: the built-in dashboards above **plus** every admin-saved dashboard. From the hub the user opens any of them; because the hub has no page context, opening a parameterized built-in dashboard **prompts for its input** (e.g. enter a team number to view that team's dashboard for an event, or pick the teams for the compare dashboard) — the same input the dedicated page would otherwise pass in for you. **Admin-created saved dashboards are season-scoped** and appear in this list for the year they belong to.

**Two kinds of statistics (Q10.9-ops, closed 2026-08-21).** The app has (a) the **dynamic robot/team statistics** built by pointing charts at form fields, and (b) **operational (meta) statistics** — data *about the scouting itself*, not robot performance. **(b) is configurable, not static:** it uses the **same dashboard builder** with the **data source switched from form fields to app metadata.** Meta **dimensions**: scouter/user, match, event, form.

**Operational meta-metric catalogue — v1 (finalized on Topic 13 close, 2026-08-21):**

- **Entries submitted per user** — throughput / participation per scouter.
- **Entries submitted per match.**
- **Entries submitted per event / total.**
- **Conflicting-entry count** — the size of the §7.3 review-conflicts queue (genuine offline divergence awaiting lead/admin resolution). There is no separate *duplicate* count: last-write-wins means non-divergent duplicates never persist.
- **Super-scouting coverage** — of the matches that have any recorded entry, how many also have a super-scout entry. Denominator is *recorded* matches, so it needs no imported schedule.

**Deferred (→ §24, gated on TBA schedule import):** schedule-relative coverage — matches scouted vs. **scheduled**, robots covered of 6 per match, teams scouted of total, qualification matches recorded vs. remaining — and the **prebuilt coverage matrix** (Topic 13). Also dropped from v1: pending-sync/outbox count and last-sync time, active-forms-this-season, and fields-per-form (low value; the admin already knows the form structure).

**Meaningful value shading (all dashboards, confirmed 2026-08-21).** Every numeric cell/mark that carries a metric value is **color-scaled from light red (worst) to light green (best)**, so a table or heatmap reads at a glance. Scaling is **per column/metric** (each metric shaded within its own domain, since units differ) and is driven by the field's `direction` semantic metadata (§3), which already has the three states needed — **no schema change**. The domain per metric:

| Metric kind | Color domain |
|---|---|
| Numeric field with a declared `expected_range` | that declared min–max |
| Unbounded numeric (e.g. game pieces, no range) | observed min–max of the **values shown in that column under the dashboard's current scope + filters** — so the same team's value can shade differently on an event dashboard vs. a season dashboard, because the reference population changed |
| Percentage / rate (success rate, etc.) | fixed **0–100 %** (absolute level matters, not rank within the set) |
| **Ordinal enum** (e.g. climb level None→Deep) | option **rank by builder list order** (top of the list = lowest rank); aggregations (avg/median across matches) run on the rank index, so "avg climb level" is a colorable, sortable number |
| `direction: neutral` | **no scale** — flat single color (values aren't ordered good→bad, e.g. start position, robot number) |

`direction: lower_is_better` **inverts** any scale (green = low), covering climb time, cycle time, fouls, tip-over rate. **Capture-time convention:** for an ordinal enum, the admin must list options **worst → best** at field creation, because that order becomes the rank — it cannot be inferred later.

**Operational meta-metrics** have no field-level `direction` or `expected_range`, so they shade against the **observed min–max of the population shown** and default to **higher-is-better** (more entries / more coverage = green) — e.g. a team with 2 recorded matches shades red, one with 10 shades green, relative to that column.

**Shading edge cases (confirmed 2026-08-21):**

- **No data** — a cell with no value for that metric renders as a **distinct grey "—"** and is **excluded from the column's min/max domain**. Missing data must never look like a bad value (the confusion §13.1 warns about).
- **All-equal / single row** — when a column's min == max (every value identical, or only one row), shading falls back to a **flat mid-color**; inferring "best" or "worst" from one data point would be dishonest.
- **Colorblindness** — the red→green scale must also vary **lightness monotonically** (so it degrades to a legible light→dark ramp for red-green colorblind users, ~8% of males) **and the numeric value is always shown in the cell.** Color is a fast cue, never the only channel.

### 10.3 Closed questions

- ~~**Q10.1**~~ ✓ **Full high-analysis chart set incl. pie/donut, plus stacked-area, bump/rank, bullet-gauge, small-multiples** (§10.1/§10.2).
- ~~**Q10.2**~~ ✓ **Shared:** any saved dashboard is visible to all users; drafts are session-private, never shared; saving admin-only (§10.1, Topic 5 §5.2).
- ~~**Q10.3**~~ ✓ **Read-only** when fields no longer exist; **re-map wizard → §24 nice-to-have.**
- ~~**Q10.4**~~ ✓ **Export → §24 nice-to-have.**
- ~~**Q10.5**~~ ✓ **Top-8 + bottom-8 on phone for all-teams charts (show-all expansion); everything shown for per-game/per-team slices; per-team match views designed for up to 14 matches.**
- ~~**Q10.6**~~ ✓ **Recharts** for standard charts; **image overlays + team×metric heatmap hand-built in SVG/Canvas**; ECharts rejected as overkill.
- ~~**Q10.7**~~ ✓ **Drill-down → §24 nice-to-have.**
- ~~**Q10.8**~~ ✓ **Scouted-vs-official → §24 nice-to-have, gated on TBA import.**
- ~~**Q10.9**~~ ✓ **Metric selector/expand pool = metrics sharing the chart's X dimension** (comparable axis/unit).
- ~~**Q10.10**~~ ✓ **Expand-to-stack capped at 4.**
- ~~**Q10.9-ops**~~ ✓ **Operational stats are configurable via the same builder over app-metadata sources, not a static page** (§10.2); meta-field catalogue finalized at Topic 13 close. *(This is the original 2026-08-18 Q10.9 — "what belongs on the general stats page" — now answered: it's builder-driven, not a fixed page.)*

---

## 11. Search, ranking & browse

### 11.1 Confirmed requirements

- Search for teams and entries, rank teams, compare teams, and preview matches — via dedicated pages (§11.2).

### 11.2 Confirmed decisions

*(Confirmed 2026-08-21 on closing topic 11.)*

**Terminology.** A **"form"** in a user's words = a submitted **entry** (one robot's record for one match), which is distinct from the form **template** (§3). This section uses **entry** for the record.

**No global omnibox** — the browse/search surface is a set of **dedicated pages** (simpler, matches how the team thinks). The old single-search-box proposal is dropped.

**Canonical basic rank.** Everywhere a generic "rank" is needed, it is the **status-aware average points per match** — the mean of a team's *played* match scores, with **no-show / disabled robots excluded, never counted as 0** (Topic 6/9). This is the default sort on every ranked view.

**Event switching is offline-disabled.** Any action that switches the active competition is **disabled offline** — Topic 7 caches only the active competition, so another event's data (and a team's cross-event history) isn't on-device. This applies to every switch entry-point, including the one on the team search page below.

**The pages:**

**1. Team search page.** Searches **all teams in the season** by number or name. Each team in the **active event** shows a **small side rank badge**; the **top 3 get a medal icon**. Teams not in the active event show no rank.
- Team **in the active event** → button → that team's **statistics page** (the Topic 10 team dashboard).
- Team **not in the active event** → a **differently-styled button** → pick from the **events that team competed at** (events where we hold entries for it) → **"are you sure?" confirmation** → on yes, **switch the active event as a session-only override** (never saved as the admin default, per §4.1 — this is logged as an additional guarded override entry-point) and land on that team's stat page. Disabled offline (see above).

**2. Entry (form) preview page.** A read-only, nicely formatted rendering of a **single entry**: all field values laid out **by phase**, plus the entry's **scouted score (points)**, team, match, scouter, form kind, and timestamp.

**3. Entry (form) search page.** Lists entries of the **active competition only**, **both match and super kinds with a kind filter**. Searchable by **team name, team number, match number, scouter name**. Each row shows the entry's **scouted points**. Row / button → the entry preview page (2).

**4. Ranking page** *(also the Topic 10 §10.2 "ranking" built-in dashboard).* Backed by a single **admin-built ranking dashboard — one per season, edited in place (no versioning)** — that defines the table's metric **columns**. Layout: the **metrics table first**, then charts / other metrics below. **Only the table's built-in metrics** can drive **column-click reordering** and be weighted. **Reliability/availability** (Q9.5) is one of those selectable, sortable, weightable columns.
- **Weighting mode** (toggle): each metric is normalized **min-max to 0–1** (for a `lower_is_better` metric the normalized value is inverted as **`1 − normalized`**, so higher is always better), each is given a **weight** (auto-normalized — they need not sum to 1), and teams are ranked by the **weighted sum**. A per-team **contribution breakdown** shows how much each metric added to the final score.
- **Missing metric for a team:** the table cell shows a grey "—"; in the weighted composite it counts as **0.5** (the neutral midpoint) so it neither sinks nor inflates the team.
- **Weight presets:** an admin can **save named weight presets** (these seed the Topic 12 pick list); a lead can adjust weights live but **session-only**; **offline, weights can be changed but not saved** (Topic 7). Normalization population = the ranked teams under the dashboard scope (the active event by default).

**5. Compare page** *(Topic 10 §10.2 "compare" built-in).* Compares **up to 6 teams** on an **admin-built compare metric set (one per season, no versioning) computed over the active event**.
- **2 teams:** a "final-score" **head-to-head scoreboard** — a big avg-points headline per team, then per-metric rows with the better value highlighted.
- **3–6 teams:** **radar + table**, designed to a high visual standard.

**6. Match preview page** *(Topic 10 §10.2 "match preview" built-in).* The **user enters the six teams** (3 red + 3 blue) — v1 has no schedule (TBA deferred, §24); TBA later auto-fills them. Shows the six teams grouped by alliance with their key metrics.
- **Prediction:** each alliance's predicted score = the **sum of its three teams' status-aware avg points/match**; the higher total is the predicted winner; shows **each team's contribution and the predicted margin**.
- Also surfaces **per-robot reliability / no-show flags** and each team's **top strengths**.

### 11.3 Closed questions

- ~~**Q11.1**~~ ✓ **CLOSED 2026-08-21:** yes — composite **weighted ranking** on the ranking page (min-max 0–1 per metric, `1 − norm` for lower-is-better, weighted sum, per-team contribution breakdown, admin-saved presets that seed Topic 12).
- ~~**Q11.2**~~ ✓ **CLOSED 2026-08-21:** full-text **notes search → §24 nice-to-have** (needs a Postgres text index + an offline story).
- ~~**Q11.3**~~ ✓ **CLOSED 2026-08-21:** **no multi-season team-history view** — out of scope.
- ~~**Q11.4**~~ ✓ **CLOSED 2026-08-21:** **TBA public info on the team page → §24 nice-to-have**, gated on the TBA import.
- ~~Global search omnibox~~ ✓ **Dropped** in favour of dedicated search pages.

---

## 12. Alliance selection & pick list

**Status: CLOSED 2026-08-31 (Q12.1–Q12.3).** §12.1 keeps the rationale; §12.2 holds the confirmed decisions; §12.3 the data model; §12.4 the page design; §12.5 the closed questions.

### 12.1 Why I'm raising this **[RAISED BY ME]**

You didn't mention it, but it's the reason scouting exists. Everything the app collects over two days is spent in a 20-minute window where a lead must respond in seconds to "team 254 just got picked, who's next?". If the app doesn't support that moment, the data doesn't get used.

### 12.2 Confirmed decisions

*(Confirmed 2026-08-31 on closing topic 12.)*

**In scope for v1 (Q12.1),** and **moved from Phase 3 to Phase 2** (§19.1). The original Phase 3 bundled the pick list with the TBA/FRC import, which is deferred to §24 — but the pick list needs nothing TBA provides (no schedule, no official results), so leaving it there would have deferred it by association.

**Ownership — the admin owns everything, with exactly one exception (Q12.3).** No collaborative editing: **single editor, many viewers.**

- **Admin only:** create the pick lists, reorder them, add/remove teams, edit or remove do-not-pick entries, and record the alliance bracket.
- **Lead:** may **add a team to the do-not-pick list with a required reason** — and nothing else. A lead cannot reorder a pick list, cannot add or remove a team on one, and cannot edit or remove a do-not-pick entry (not even one they created).
- **Scouter and Lead:** read-only view of the pick lists, the do-not-pick list and the alliance bracket (consistent with §5.1 — all data is visible to every role).

**Two ordered pick lists per event: `first` and `second`.** Different criteria justify different lists (a first pick complements you; a second pick is often defensive or specialised). Each is an ordered list of teams built by **drag-reorder** and **seeded from a ranking weight preset** (§11.2): the admin picks a saved preset (or the ranking page's current live weights) and the list is generated in that weighted-composite order, then hand-adjusted. **Reseeding replaces the whole order and asks for confirmation first**, because it discards manual adjustments.

Each row shows: **rank position**, team number + name, the **canonical basic rank** (status-aware avg points/match, §11.2), the **reliability/availability counts** (breakdowns / no-shows / disabled, §9.2), the **seeding preset's metric columns**, and the team's **inline notes**.

**Second-pick round awareness.** The app tracks which round selection is in. While any alliance still has an empty `pick1` slot the **first-pick list** is the active list; once all 8 alliances have a first pick the app **switches to the second-pick list automatically** (manually overridable both ways). Only the active list is emphasised; the other stays viewable.

**Do-not-pick list — its own list, reason required, blocking by default.**

- Adding a team **requires a free-text reason** — no reason, no add. The author is recorded on the entry.
- **A team on the do-not-pick list cannot be added to a pick list.** The add action is **blocked**, showing the reason and who wrote it. To pick that team you must first remove them from the do-not-pick list.
- **Exception — the team is *already* on a pick list when it gets do-not-picked:** this is **flagged, never blocked**. The team stays exactly where it is in the order, is rendered with a warning marker, and a banner tells the admin to resolve it. A lead's addition therefore never fails and never silently reorders the admin's list.
- **One-tap removal, from the do-not-pick page itself.** Every row carries a remove action in place (inline confirm — no dialog chain, no navigating to another screen). Removing an entry **clears any pick-list flag it caused, immediately**.

**Alliance bracket — manual entry, synced when possible (Q12.2).** No import: TBA is deferred (§14 → §24) and the official feed lags by minutes, which is useless in a room where a pick happens every 30 seconds.

- **8 alliances × 3 robots** — `captain`, `pick1`, `pick2` — plus an optional **`backup`** slot per alliance.
- The admin enters each result manually as selection happens. Every entry **writes locally first** (optimistic local UI, §8.1) and **syncs immediately if there is internet, otherwise on the next sync** — the ordinary §7.3 outbox path. This matters because selection is watched from the stands, where there is usually no signal.
- **Declined pick.** A team that declines is recorded as **declined against that alliance**. It stays available to be picked later (or to become a captain) per FRC rules, and the app shows a "declined — alliance N" marker so nobody asks twice or wastes a pick.
- **Live cross-off.** As captains, picks, and backups are entered, every affected team is **struck through automatically across both pick lists**, so the top unstruck row is always the next available team.

**Cross-off propagation, with realtime deferred (§8.1).** On the **admin's own device** cross-off is instant (local optimistic write). **Viewers online** see it on the standard **45-second auto-refresh** or a manual refresh. **Viewers offline** see nothing until they sync. The operating rule is therefore explicit: **during selection the admin's device is the source of truth**, and every other screen is advisory. If cross-device realtime lands later (§24) it upgrades this with no design change.

**Offline behaviour.** Pick lists, the do-not-pick list and the alliance bracket are all **editable offline** and sync through the outbox. This **extends §7.3**, which previously said only "the admin-only pick list" — it now covers all three surfaces, and records that a **lead may add a do-not-pick entry offline** too.

**Reordering is one operation against a list-level version. [decided-by-Claude]** §7.3's last-write-wins is a *per-row* rule, but a pick list is an **ordering** — two devices reordering the same list offline would silently discard one person's entire reordering. So the `pick_lists` row carries **its own version**, and a drag-reorder writes the **whole ordering as a single operation** stamped with the base version it started from. Fast-forward → applied and the version bumps. Genuine divergence → the §7.3 **review-conflicts** queue with the superseded ordering preserved. Individual team rows (add / remove / edit note) remain ordinary last-write-wins rows. Single-editor ownership makes this rare; the guard exists so that when it happens, nobody loses an hour of ordering work.

**Printing stays deferred.** A printed pick list for the pit is standard practice, but it rides on the deferred export work (Q10.4 → §24), so **v1 is screen-only**. **Q16.4** (printable views) remains open and is where this returns. **[RAISED BY ME]**

**No pick-list history and no ordering snapshots** — consistent with "no per-entry edit history" (§13.2) and "no user audit log" (§5.1). The bracket's `declined` markers are the only historical record the feature keeps.

### 12.3 Data model

```
pick_lists(id, event_id, kind ∈ {first, second}, seeded_from_preset_id?, version, updated_at)
pick_list_entries(id, pick_list_id, team_id, rank, note?, deleted_at?)

do_not_pick(id, event_id, team_id, reason /* required */, created_by, created_at, deleted_at?)

alliances(id, event_id, number 1..8)
alliance_slots(id, alliance_id, slot ∈ {captain, pick1, pick2, backup}, team_id?, deleted_at?)
alliance_declines(id, alliance_id, team_id, created_at)
```

`rank` is a per-list integer rewritten **wholesale** by a reorder operation (§12.2); `version` exists on `pick_lists` only — it is the ordering guard. Everything is scoped to one **event** (§4.1), client-UUID keyed, soft-deleted (`deleted_at`), and rides the §7.3 outbox. Cross-off state is **derived** from `alliance_slots`, never stored on the pick list — so a corrected bracket instantly corrects every list (same principle as derived scores, §2.2).

### 12.4 The pages

One **"Alliance selection"** area with three tab-switched pages:

1. **Pick list page** — the active-round list (first/second toggle plus automatic round detection), drag-reorder for the admin, automatic strikethrough for taken teams, do-not-pick warning markers, inline notes, a reseed-from-preset action, and a **compact "next available team" header** designed to be readable at arm's length in a loud arena.
2. **Do-not-pick page** — every flagged team with its reason and author, a **one-tap remove** per row, and an "add team" action available to leads as well as the admin.
3. **Alliance bracket page** — the 8 alliances × captain / pick1 / pick2 / backup grid; tap a slot to enter a team; declined markers shown against the alliance that was refused.

### 12.5 Closed questions

- ~~**Q12.1**~~ ✓ **CLOSED 2026-08-31:** **in scope for v1**, and **moved from Phase 3 to Phase 2** (§19.1) — it depends on nothing from the deferred TBA import.
- ~~**Q12.2**~~ ✓ **CLOSED 2026-08-31:** alliance-selection results are **entered manually**; each entry writes locally and **syncs immediately when online, otherwise on the next sync** (§7.3 outbox). No API import.
- ~~**Q12.3**~~ ✓ **CLOSED 2026-08-31:** **no collaborative editing** — the **admin edits, everyone else views**. Single exception: a **lead may add a do-not-pick entry with a required reason**, and may not edit or remove one.
- ~~Live cross-off assumes realtime~~ ✓ **Resolved:** cross-off is instant on the admin's device, reaches online viewers on the **45 s auto-refresh**, and offline viewers on sync; the admin's device is the declared source of truth during selection (closes the §8.2 dependency note).

---

## 13. Data quality, integrity & scouter reliability

### 13.1 Why I'm raising this **[RAISED BY ME]**

Scouting apps don't fail because of bad charts. They fail because 30% of the matches are missing, one scouter recorded the wrong robot, and someone entered 47 game pieces when the maximum possible is 9 — and nobody noticed until the pick list was being made. Detection has to be part of the product.

### 13.2 Confirmed decisions

*(Confirmed 2026-08-21 on closing topic 13.)*

- **Entry-time validation = hard range block only** (Q13.4). Submission is **blocked** only when a numeric value falls **outside the field's declared `expected_range`** (min/max set at field-creation time, §3). Fields without an `expected_range` have no block. **No cross-field rules and no outlier blocking** in v1 — reality is weirder than any rule, and a hard cross-field rule risks losing a legitimately unusual match.
- **Outlier / distribution flagging → §24 nice-to-have** (Q13.1). Advisory warnings for values far outside a team's or the event's distribution are wanted but deferred; nothing is ever auto-excluded or silently averaged out.
- **Coverage matrix → §24 nice-to-have, gated on TBA schedule import** (Q13.1). The matches × robot-stations grid cannot mark a cell "missing" without a master schedule, and v1 has no imported schedule (§14 → §24). It returns when TBA import lands.
- **No redundant-scouting feature** (Q13.2). Deliberate double-scouting of the same robot and agreement measurement are **not** built. If two entries for the same `(team, match)` ever exist (offline divergence), they are handled by the **existing §7.3 path**: last-write-wins live, and only genuine divergence is surfaced to the lead/admin **"review conflicts"** queue with the superseded version preserved. The **conflicting-entry count** meta-metric (§10.2) is simply the size of that queue.
- **No scouter reliability score** (Q13.3). Not computed, not displayed, not used for weighting — the team considers it socially counter-productive.
- **No dedicated bulk-fix tools** (Q13.5). The two operations that actually happen are covered by existing mechanisms: **reassigning an entry to the correct team = editing the team on that entry** (normal edit, within §5.2 edit permissions); **merging duplicates = resolving the §7.3 review-conflicts item**. Bulk match-number shifting and shift-deletion are **out of scope for v1**.
- **No full per-entry edit history** (resolves the dangling Q5.5 cross-reference). Routine edits **overwrite** last-write-wins; only §7.3 divergence preserves the superseded copy. This is consistent with "keep the last" (Q13.2) and with the no-audit-log decision (Q5.5).

**Net effect:** Topic 13 adds **no new v1 surface** beyond the hard range-block at entry (which rides on the already-required `expected_range`) and the **operational meta-metrics** finalized in §10.2. The richer integrity tooling (coverage matrix, outlier flags) is real but deferred behind TBA import.

### 13.3 Closed questions

- ~~**Q13.1**~~ ✓ **CLOSED 2026-08-21:** coverage matrix **and** outlier flagging → **§24 nice-to-have** (coverage matrix gated on TBA schedule import).
- ~~**Q13.2**~~ ✓ **CLOSED 2026-08-21:** no redundant-scouting feature; conflicts handled by the existing §7.3 review-conflicts path (keep last, preserve superseded).
- ~~**Q13.3**~~ ✓ **CLOSED 2026-08-21:** no scouter reliability score.
- ~~**Q13.4**~~ ✓ **CLOSED 2026-08-21:** hard block only on out-of-`expected_range` values; outlier warnings are advisory and deferred (§24).
- ~~**Q13.5**~~ ✓ **CLOSED 2026-08-21:** no dedicated bulk-fix tools; reassign = edit the entry's team, merge = resolve a review-conflict.

---

## 14. External data integration (TBA / FRC Events API)

### 14.1 Why I'm raising this **[RAISED BY ME]**

Manually typing 40 team numbers and 100 matches into an app at 8am on competition day is both miserable and error-prone, and it's unnecessary: **The Blue Alliance API** and the official **FRC Events API** provide the event list, team list, match schedule, and match results for free. This one integration removes most manual setup and enables schedule-driven assignment (topic 6) and result validation (topic 13).

### 14.2 Proposed decisions

- Server-side integration (API keys never in the browser) with a scheduled/triggered sync that imports: seasons, events, teams per event, match schedules, match results, and optionally team awards and photos.
- Imported data is stored locally in our own tables, so the app works offline and doesn't depend on a third party being reachable at the venue. Sync runs beforehand, at the hotel.
- A manual fallback: paste/upload a schedule, or enter matches by hand, in case the API is unavailable or you're at an unofficial scrimmage.

### 14.3 Open questions

- ~~**Q14.1** — Do you want this integration? TBA, the official FRC Events API, or both?~~ ✓ **CLOSED 2026-08-14: wanted, deferred → §24 nice-to-have.**
- **Q14.2** — Do you have (or can you get) a TBA read API key? It's free — you request it from your TBA account.
- **Q14.3** — Should match **results** be imported (scores, ranking points, winners), or only the schedule?
- **Q14.4** — How often should it sync during an event, given connectivity is limited? Manual "sync now" only, or automatic when online?
- **Q14.5** **[RAISED BY ME]** — Schedules **change** at events (replays, surrogate matches, delays). Do you need to handle a re-imported schedule that conflicts with data you've already collected against the old one?

---

## 15. System architecture, repo layout & services

### 15.1 Confirmed requirements

- One repository, separate client and server projects, deployed as separate services.
- Supabase as the database.
- Deployment on Vercel.
- **TypeScript everywhere** (Q15.2, Q15.4). Client: **React + Vite** PWA. Server: **Node + Hono**. A shared `packages/shared` holds the metric engine, form-definition model and Zod schemas so the offline (browser) and server code paths are literally the same code. *(confirmed 2026-08-14)*
- **All traffic goes through the server API** (Q15.1) — no direct client→Supabase access on the hot path. Load is low, and this is clean now that cross-device realtime is deferred (topic 8, §24). *(confirmed 2026-08-14)*
- **Transport-agnostic use-case layer** (Q15.8): every operation is a named, typed, described use case (Zod in/out) in `packages/shared` / `server/core`; HTTP is the transport now, MCP mappable mechanically later. *(confirmed 2026-08-14)*
- **Dynamic-form validation is generated at runtime from the field definitions.** Hand-written Zod covers only the fixed skeleton and the use-case tool inputs/outputs; a new season's form needs no code change. *(confirmed 2026-08-14)*

### 15.2 Proposed decisions

**Repository layout** (monorepo with pnpm workspaces + Turborepo):

```
frc-scouting/
  apps/
    client/          # the PWA — React + Vite + TypeScript
    server/          # API service — Node + Hono (or Express) on Vercel Functions
  packages/
    shared/          # TypeScript types, Zod schemas, form-definition model,
                     # metric evaluation engine (shared by client offline + server)
    db/              # Supabase migrations, generated DB types, seed data
  docs/
    spec/            # this document
    plans/           # implementation plans
```

A shared package is important here: the form model, the validation rules, and the metric engine must behave identically online and offline, and duplicating them in two languages guarantees drift.

**What is the server actually for?** *(Resolved — Q15.1.)* **All traffic goes through the server API; the client never calls Supabase directly.** The server (service role) owns every path:

| Path | Used for |
|---|---|
| Client → Server → Supabase (service role) | **Everything** — reads, scouting-entry writes, form creation/versioning and view generation, user provisioning and role changes, exports, aggregations, and (deferred) TBA/FRC imports. |

The earlier **hybrid** proposal (client → Supabase directly for reads + **realtime subscriptions**) was rejected. Its one real advantage was Supabase Realtime, and cross-device realtime is now **nice-to-have** (§8, §24). A single control point gives cleaner authorization (§5.2 use-case-layer checks, no RLS), easier auditing, and the mechanical MCP mapping. **Optimistic local UI stays** — the client applies its own writes instantly and reconciles on server confirm (§8.1) — and a **45-second refresh** covers cross-device propagation without WebSockets.

**Transport-agnostic service layer — required by the MCP goal. [RAISED BY ME]**

Because stage two is exposing this data to an LLM over MCP, the server must not be written as a pile of HTTP route handlers with logic inside them. Instead, every meaningful operation becomes a named, typed **use case** in `packages/shared` or a `server/core` module:

```
core/
  queries/    getTeamStats, rankTeams, compareTeams, getMatchSchedule,
              queryEntries, getCoverageReport, searchNotes, getFormDictionary
  commands/   createForm, publishFormVersion, upsertEntry, setPickList, importEvent
```

Each one has a Zod input schema, a Zod output schema, and a plain-language description. Three thin transports then sit on top of the same functions:

| Transport | Consumer |
|---|---|
| HTTP / tRPC | the web client |
| MCP tools | an LLM (Claude Desktop, Claude Code, or your own in-app agent) |
| CLI / scripts | imports, backfills, testing |

The pay-off is that MCP support becomes a mechanical mapping — a Zod schema converts directly to the JSON Schema an MCP tool advertises, and the description you already wrote becomes the tool description the model reads. If we don't do this, MCP means writing a second, parallel implementation of every query, and the two will drift.

**Constraints to design around on Vercel:** serverless functions are stateless and time-limited, so no long-running import jobs — chunk them, or run them as Vercel Cron invocations. No local filesystem persistence. Cold starts exist, so don't put the scouter's critical path behind a rarely-called function.

### 15.3 Open questions

- ~~**Q15.1** — Do you accept the **hybrid** access model, or all traffic through the server API?~~ ✓ **CLOSED 2026-08-14: all traffic through the server API** (§15.1).
- ~~**Q15.2** — Client framework: Vite + React SPA or Next.js?~~ ✓ **CLOSED 2026-08-14: React + Vite PWA** (§15.1).
- **Q15.3** — Server framework: Hono (tiny, fast, great on Vercel Functions), Express (familiar), or Next.js API routes? And do you want **tRPC** for end-to-end typed calls between client and server?
- ~~**Q15.4** — TypeScript everywhere?~~ ✓ **CLOSED 2026-08-14: yes** (§15.1).
- **Q15.5** **[RAISED BY ME]** — Do you want **generated database types** from Supabase (so a schema change surfaces as a compile error), and **Zod** validation shared between client and server?
- **Q15.6** **[RAISED BY ME]** — Where do **file uploads** (robot photos) go — Supabase Storage, presumably? And who is allowed to upload?
- **Q15.7** **[RAISED BY ME]** — Do you need a **local development** setup that works without touching production data (Supabase local via Docker, or a separate dev project)? I strongly recommend a separate dev Supabase project at minimum — you do not want to test a "delete form" feature against real competition data.
- ~~**Q15.8** **[RAISED BY ME]** — Do you accept the **use-case / transport-agnostic service layer**?~~ ✓ **CLOSED 2026-08-14: yes** (§15.1).
- **Q15.9** **[RAISED BY ME]** — Should the **MCP server be its own deployable app** (`apps/mcp`) or a route inside the existing server app (`apps/server/mcp`)? A separate app gives independent scaling, its own auth surface and cleaner logs; a route is less to maintain. I lean toward a route in the server app now, extractable later, since they share the same core.

---

## 16. UI/UX design system & responsive behaviour

### 16.1 Confirmed requirements

- Must look good and work well on both phone and computer screens.
- **App chrome is English and LTR** (kept simple); **form content is Hebrew** — labels and free-text notes (Q16.2). Full-app RTL is **not** required, but every text node that can hold Hebrew (form labels, notes, and their appearance on chart axes, tables and team cards) must be bidi-correct (`dir="auto"`). Built in from the start, not retrofitted. *(confirmed 2026-08-14)*
- **The active-context (season/event) switcher lives on a dedicated context/landing page, not in the persistent header/nav** — so the working context can't be changed by accident mid-task (§4.1). *(confirmed 2026-08-17)*

### 16.2 Proposed decisions

- **Two distinct experiences, one codebase**: *phone = data entry* (huge touch targets, minimal chrome, one task per screen, thumb-reachable actions); *desktop = analysis* (dense tables, multi-panel dashboards, keyboard shortcuts, hover detail). These are not the same UI scaled — they're different information densities, and I'd design them as such.
- Tailwind CSS + a component library (shadcn/ui) for speed and consistency.
- **Dark mode**, defaulting to dark: scouting happens in dim arenas, and it saves phone battery.
- Layout: mobile-first, with breakpoints at roughly phone / tablet-landscape / desktop.
- Loading states everywhere, and skeletons rather than spinners for lists.
- An unmistakable, always-visible **offline/sync indicator** — this is a primary UI element, not a footnote.

### 16.3 Open questions

- **Q16.1** — Do you have **team branding** (colours, logo, team number) to build in?
- ~~**Q16.2** — **Language**: English only, Hebrew only, or both?~~ ✓ **CLOSED 2026-08-14: English LTR chrome, Hebrew bidi-aware form content** (§16.1). **[RAISED BY ME]**
- **Q16.3** — Any app you like the look/feel of that I should use as a reference?
- **Q16.4** **[RAISED BY ME]** — Do you need **printable views** (pick list, match preview, blank paper backup form)? A printed paper form as an emergency fallback when the tablets die is standard practice.
- **Q16.5** **[RAISED BY ME]** — Should the app be usable **with gloves / wet hands / in sunlight**? Affects target sizes and contrast.
- **Q16.6** **[RAISED BY ME]** — Do you want the field diagram for position-picking fields to be **uploadable per season** (an image you upload each year), so this stays year-agnostic? I think yes.

---

## 17. Non-functional requirements

### 17.1 Proposed targets (challenge these)

| Area | Target |
|---|---|
| Cold start offline | App interactive in under 3 seconds on a mid-range phone with no network. |
| Data entry responsiveness | Every tap registers visually in under 100 ms, always local-first. |
| Sync latency | A submitted entry appears on other online devices within 2 seconds. |
| Scale | ~10 events/season, ~50 teams/event, ~120 matches/event, ~6 entries/match, ~80 fields/entry → roughly 6,000 entries and 500,000 field values per event. Multi-season: low hundreds of thousands of rows. This is small for Postgres, which is why Option A in topic 3 is safe. |
| Database budget | **Supabase free tier is the operating constraint.** Store raw entries only — never persist derived scores or aggregates (the shared engine computes them); no photo/pit data in v1. Keep DB actions simple: plain SQL views (not materialized), no heavy triggers or DB-side cron. Escalate before approaching the size cap. *(confirmed 2026-08-17, Q17.3)* |
| Concurrent users | (Depends on Q1.1.) |
| Security | Authorization enforced in the server use-case layer and surfaced in the UI — **no Postgres RLS / per-row policies** (Topic 5); no service-role key in the browser; **no user audit log** (Q5.5). |
| Backup | Automated export of all data per season; the ability to restore. |
| Testing | Unit tests for the metric engine and sync logic; integration tests for the sync protocol; the offline path must be tested, since it's the one that's hardest to debug at a venue. |

### 17.2 Open questions

- **Q17.1** — Are those performance targets right, and is the data-scale estimate roughly correct for your team?
- **Q17.2** **[RAISED BY ME]** — What's your **backup and export** requirement? At minimum I'd want a one-click "export this event/season to CSV+JSON" so a weekend's work can never be lost to a bug, an accidental delete, or a Supabase free-tier surprise.
- ~~**Q17.3**~~ ✓ **CLOSED 2026-08-17: Supabase free tier.** Design lean per the Database-budget row (§17.1). Two free-tier risks and their mitigations: **(1) inactive-project pause** — offline-first means scouting never depends on the DB being awake; wake and verify the project before each event, and exports (Q17.2) insulate the data regardless; **(2) size/egress caps** — raw-only storage keeps a season well within budget, but multi-season growth is archived out (export + prune) rather than kept live. If we ever must exceed the free tier I'll flag it; a plain-Postgres alternative (e.g. Neon free tier, or self-hosted Postgres) is a low-friction migration because topic 3 uses no Supabase-specific runtime DDL.
- **Q17.4** **[RAISED BY ME]** — Privacy: minors' names in the database. Do you need anything specific here (initials only, no emails, deletion on graduation)?
- **Q17.5** **[RAISED BY ME]** — How much automated testing do you want? Real answer, not aspirational — it changes how the implementation plan is structured.

---

## 18. Deployment, environments & operations

### 18.1 Confirmed requirements

- Client and server both deployed on Vercel.

### 18.2 Proposed decisions

- Two Vercel projects from one repository, each with its own root directory and build command, with Turborepo caching.
- Environments: **production**, **preview** (per pull request), **local**. Separate Supabase projects for production and development.
- Database migrations live in the repo (`packages/db`) and are applied via the Supabase CLI in CI — never by hand-editing the production schema in the dashboard.
- Environment variables: only the Supabase anon key and URL reach the client; the service-role key and external API keys live in the server project only.
- Basic error tracking so failures at a competition aren't invisible.

### 18.3 Open questions

- **Q18.1** — Do you have a **domain**, or is a `*.vercel.app` URL fine? (A short, memorable URL matters when 20 people are typing it on phones in a pit.)
- **Q18.2** — Do you want **CI** (typecheck, lint, tests, migration check on every push)?
- **Q18.3** **[RAISED BY ME]** — Do you want error/usage monitoring (Sentry, or Vercel's built-in analytics)?
- **Q18.4** **[RAISED BY ME]** — Is there any scenario where you need this to run **fully self-hosted or on a laptop at the venue** as a last resort? That's a significant architectural constraint if yes, so better to know now than to discover it at an event.
- **Q18.5** **[RAISED BY ME]** — Who holds the Vercel and Supabase accounts — a personal account, or a team account that survives students graduating? Handover has killed more team tools than bugs have.

---

## 19. Delivery phases & priorities

### 19.1 Proposed phasing

Everything above at once is a very large build. Suggested order, where each phase is independently useful:

**Phase 1 — Core loop (must have before any event)**
Auth and roles → events and teams → form builder with core field types → offline data entry → sync → **QR fallback transfer (animated + compressed)** → raw data browse → basic per-team stats and ranking table.

**Phase 2 — Analysis**
Metric builder → chart and dashboard builder → team compare → **pick list, do-not-pick list & alliance bracket** → coverage/quality matrix → export. *(Alliance selection moved up from phase 3 on Topic 12 close — it depends on nothing the TBA import provides.)*

**Phase 3 — Competition workflow**
TBA/FRC API import → scouter assignments → schedule-driven coverage matrix and official-result validation. *(Pick list and alliance selection moved to phase 2 on Topic 12 close, 2026-08-31.)*

**Phase 4 — Depth**
Pit and super scouting forms → photos → advanced field types (position picker, event log, cycle times) → scouter reliability → multi-season history.

**Phase 5 — MCP server (DEFERRED, not currently in scope)**
Expose the existing use cases as MCP tools, resources and prompts → auth for the MCP endpoint → connect an MCP client and iterate on tool descriptions → evals.

**Phase 6 — In-app AI (NICE-TO-HAVE, see §24)**
Server-side LLM orchestration → in-app chat/insights panel → cached briefing documents → notes summarisation → natural-language-to-chart. *(Moved to §24 nice-to-have 2026-08-15. MCP server, phase 5, remains parked in topic 20 — not nice-to-have.)*

**Important:** phases 5 and 6 are deferred by the scope decision in topic 20.1, but two of their prerequisites are *early* and cannot be deferred — the **semantic field metadata** in topic 3.3 (part of phase 1's form builder) and the **use-case service layer** in topic 15.2 (part of phase 1's server). Everything else about AI can wait; those two cannot.

### 19.2 Open questions

- **Q19.1** — Does this phasing match your priorities? What would you move earlier?
- **Q19.2** — What is the **first thing you want to be able to do** in a working app? That defines phase 1's true boundary.
- **Q19.3** — Who is building this — you alone, you plus Claude Code, or a team of students? And roughly how much time per week?

---

## 20. AI insights, LLM integration & MCP readiness

### 20.1 Confirmed requirements

- The backend must be **built now so that connecting an LLM later is straightforward**, not a rewrite.
- Specifically, the server should be prepared to work with **MCP (Model Context Protocol)**.
- The purpose is to **get insights** from the scouting data using an LLM.

**Scope decision (confirmed 2026-08-12): setup only. No LLM connection is in scope.**

We build the *foundations* that make MCP possible and we stop there. Concretely, in scope now:

| In scope now | Where it lives |
|---|---|
| Semantic metadata on every form field | Topic 3.3 — part of the phase 1 form builder |
| Transport-agnostic use-case layer with Zod input/output schemas and descriptions | Topic 15.2 — part of the phase 1 server |
| Aggregation performed in the backend, never by a caller | Topic 9 — the metric engine, needed anyway |
| Bounded, paginated, pre-aggregated responses from every query use case | Topic 15.2 — good API design regardless |
| `include_in_ai_context` flag stored per field (unused for now) | Topic 3.3 — one nullable column, free to add |

Explicitly **not** in scope now: any MCP endpoint, any MCP transport or auth, any LLM API call, any in-app AI panel, any prompt engineering, any token budget, any AI-generated insight.

The point of this decision is that none of the deferred work requires changing anything built in phase 1. The two prerequisites above are the only things that would be painful to add later — everything else is additive.

*(2026-08-15) The **in-app AI panel** (approach B in §20.3, phase 6) is now formally logged as **nice-to-have** in §24. The **MCP server** (approach A, phase 5) remains **parked** here — its timing is undecided pending a separate decision, distinct from nice-to-have.*

### 20.2 What "MCP-ready" actually means

MCP is a standard way for an LLM to discover and call your capabilities. An MCP server advertises three kinds of thing:

| MCP concept | What it is | In this app |
|---|---|---|
| **Resources** | Read-only documents the model can load into context | The form's field dictionary for a season, the team list, an event summary, a team's stat sheet |
| **Tools** | Typed functions the model can call | `get_team_stats`, `rank_teams`, `compare_teams`, `query_entries`, `get_coverage_report`, `search_notes` |
| **Prompts** | Reusable templates a human picks | "Scouting report for team X", "Strategy brief for match N", "Explain this pick list" |

So being MCP-ready is mostly **five concrete properties of the backend**, and every one of them is worth having even if you never connect an LLM:

1. **A named, typed, described use-case layer** rather than logic inside HTTP handlers (see topic 15.2). A Zod schema becomes an MCP tool's JSON Schema for free.
2. **A machine-readable data dictionary** for the dynamic forms (see topic 3.3 semantic metadata). This is the piece that makes a year-agnostic app comprehensible to a model, and the piece that cannot be added retroactively.
3. **Aggregation done in the backend, never by the model.** The LLM must not do arithmetic on raw rows — it calls the metric engine (topic 9) and interprets the result. This keeps numbers correct and keeps token usage sane.
4. **Compact, bounded, deterministic tool responses.** A tool that returns 6,000 raw entries is useless; every tool needs pagination, sensible defaults, and pre-aggregated output. Tools return *answers*, not *dumps*.
5. **Permission-aware, read-only-by-default access.** The model's access must be scoped by the same role model as a human's (topic 5), and it must not be able to silently mutate competition data.

### 20.3 Proposed decisions

**Build the MCP server before building any in-app AI.** There are three ways to arrange this, and the order matters:

| Approach | How it works | Trade-off |
|---|---|---|
| **A. MCP server only** *(recommended first)* | Your server exposes an MCP endpoint. You connect Claude Desktop / Claude Code to it and ask questions there. | Cheapest by far — you pay no token costs, you get a full-featured chat client for free, and you learn which tools are actually useful before designing UI around them. No in-app experience. |
| **B. In-app AI panel** | The server acts as an MCP *client*, calls an LLM API, orchestrates the tool loop, streams answers into your UI. | Best experience, available to every student. You pay per token, you manage API keys and rate limits, and serverless timeouts constrain agent loops. |
| **C. Both** | Same core tools, two consumers. | The end state. B is a thin layer once A exists. |

Recommendation: **A in phase 5, B in phase 6.** A is a few days of work on top of a good service layer and immediately gives you real insights; it also tells you what B's UI should even look like.

**Transport:** MCP over **Streamable HTTP** (works on Vercel functions). The stdio transport is for local processes and is not applicable to a hosted server.

**Read/write split:** the MCP surface is **read-only in v1**. Any write capability (create a chart, set the pick list, flag an entry) is added later as a tool that *proposes* an action requiring explicit human confirmation in the app — never a direct mutation.

**Grounding and citations.** Every insight must be traceable: if the model says "1577 is your best second pick", the answer should carry the metric values and the match numbers it used. Untraceable AI claims about competition data are worse than no AI, because a student will act on them. Tools should therefore return identifiers alongside numbers so citations are possible.

**Where the LLM is genuinely better than a chart** — worth knowing, because it decides which tools matter most:

- **Summarising free-text notes.** 200 scouter comments across an event, turned into five themes per team. Charts cannot do this at all, and it's the single highest-value use.
- **Natural-language querying.** "Which teams scored more than 5 in auto and never broke down?" → resolved to a `query_entries` / `rank_teams` call.
- **Narrative match strategy briefs.** Six robots' stats turned into a paragraph the drive team can read in 90 seconds.
- **Explaining a pick list.** Turning a weighted composite score into a rationale you can defend to your mentors.
- **Anomaly narration.** "Team 4590's scoring dropped 60% after match 40 — three scouters mention a broken intake."
- **Building things from a description.** "Chart average auto pieces by team for this event" → emits a chart config in the topic 10 schema. Reuses infrastructure you're already building.

**Offline reality:** AI features require internet, so they are inherently unavailable at most of a competition. Generated insights should therefore be **saved and cached** (a stored "briefing" document per match or per team) so they can be read offline after being generated at the hotel.

**A third, much lighter AI surface: an in-app usage/help agent (parked).** Separate from both the MCP server (A) and the data-insights panel (B), a **usage agent** answers questions about *operating the app* — "how do I create a field?", "where's the pick list?", "how do I set an ordinal enum's order?" — not questions about scouting data. It is far cheaper than B because it needs **none** of the data machinery: no MCP endpoint, no access to the use-case layer or the scouting database, no semantic field metadata, and **no RAG/vector store** — the app is small enough that a single maintained **help corpus (one markdown file)** fits directly in the system prompt. Its only real prerequisite is the same deferred decision as everything else here — **an LLM connection on the server** (API key in server env, never client). Scope it to **refuse data questions** so it stays a documentation assistant. Like B, it needs internet, so it is a **build-season / at-home** aid, not a venue tool — which is fine, since usage questions arise while setting the app up, not during a match. Estimated effort once an LLM connection exists: **a day or two** (chat panel + help.md in a prompt); the ongoing cost is keeping the help doc current, which fits the mentor-maintenance model (§1.1) and changes little year to year because usage docs describe the *builder*, not a specific game.

### 20.4 Open questions

**Live now — these affect phase 1 and need answers:**

- **Q20.5** **[RAISED BY ME]** — **Privacy at the boundary.** Scouting data contains minors' names as scouters, and free-text notes that may name people or be uncharitable about other teams. Even with no LLM connected, decide the default now so the use-case layer is built to honour it: should query use cases strip scouter identities in their output by default, with an explicit opt-in to include them? I recommend yes — it costs nothing now and means no future integration can leak them by accident.
- **Q20.6** **[RAISED BY ME]** — **Reserved token/role for a non-human caller.** Do you want the role model in topic 5 to include a `service` or `agent` role from the start (read-only, scoped, no write access), even though nothing uses it yet? One extra role now avoids reworking every security policy later. Low cost, recommend yes.

**Parked until you decide to connect an LLM.** Recorded so we don't re-derive them, but not to be answered now:

- **Q20.1** *(parked)* — MCP server first, or in-app AI chat panel first?
- **Q20.2** *(parked)* — Which insights matter most: notes summarisation, natural-language querying, match strategy briefs, pick list rationale, anomaly narration, natural-language-to-chart?
- **Q20.3** *(parked)* — Who gets AI access: admins and leads only, or every scouter?
- **Q20.4** *(parked)* — Which model provider, and whose API key?
- **Q20.7** *(parked)* — Should the AI eventually take actions, or stay read-and-advise? *(Partially answered by the setup-only decision: the use-case layer separates queries from commands, so either remains possible.)*
- **Q20.8** *(parked)* — Cache and version generated insights as documents, or generate fresh each time?
- **Q20.9** *(parked)* — Build evals with a fixed test dataset?
- **Q20.10** *(parked)* — Share the MCP server with other teams? Relates to Q1.4.
- **Q20.11** *(parked)* — Give the model access to official results and TBA data alongside your scouting data?
- **Q20.12** *(parked)* — Monthly AI spend ceiling?
- **Q20.13** **[RAISED BY ME]** *(parked)* — **Usage/help agent** (see §20.3): do you want it, and at what priority relative to the data-insights panel (B)? It is cheaper and independently useful, so it could ship first once an LLM connection exists.
- **Q20.14** *(parked)* — Where does the **help corpus** live and who maintains it — one in-repo markdown file the mentor edits each season?
- **Q20.15** *(parked)* — Does the usage agent share the **same LLM connection/provider/key** as the data panel, or use a separate, cheaper/smaller model (its questions are simple)?

---

## 21. Decision log

Decisions get recorded here as topics close, so Claude Code (and future you) can see not just what we chose but why.

| Date | Topic | Decision | Rationale |
|---|---|---|---|
| 2026-08-31 | 12 / 5 / 7 / 8 / 19 | **Topic 12 CLOSED.** Alliance selection is **in scope for v1 and moved from phase 3 to phase 2** (Q12.1) — nothing in it depends on the deferred TBA import. **Admin-only editing, everyone else views** (Q12.3); the single exception is a **lead adding a do-not-pick entry with a required reason**, which they may not then edit or remove. Two ordered pick lists (**first** / **second**) per event, drag-reordered and **seeded from a §11.2 saved weight preset**, with **automatic round switching** once all 8 alliances hold a first pick. **Do-not-pick blocks an add outright, but only *flags* a team already on a pick list** — a lead's addition never fails and never reorders the admin's list — and every do-not-pick row has **one-tap removal** that instantly clears the flag. **Alliance bracket entered manually** (Q12.2): 8 alliances × captain/pick1/pick2 + optional **backup**, plus **declined** markers; writes locally and syncs now if online, otherwise on the next sync. **Cross-off is derived** from the bracket and is instant on the admin's device, 45 s for online viewers, sync-time for offline ones — **the admin's device is the declared source of truth during selection** (resolves the §8.2 dependency note). A **list-level `version` guards reorders** as one whole-ordering operation. Printing stays deferred (Q10.4 → §24; Q16.4 open). Role matrix (§5.2), offline-editable set (§7.3) and phasing (§19.1) amended to match. | The 20-minute selection window is the whole point of the two days of scouting, so the design optimises for one person answering "who's next?" in seconds, not for consensus tooling. **Single-editor ownership is the load-bearing choice:** with realtime deferred (§8) and the room usually offline, a shared editable list would produce two devices disagreeing at the worst possible moment — so one device is authoritative and everyone else reads. The lead's do-not-pick exception exists because the people who *notice* a robot is unpickable are the leads watching matches, and the cost of them being able to add a warning is far lower than the cost of that warning never reaching the list. Blocking an add but merely flagging an existing entry keeps the two roles from fighting: a lead can always raise a concern, but only the admin ever changes the order. Deriving cross-off from the bracket instead of storing it means a mistyped pick is corrected in one place and every list fixes itself, the same reason scores are derived from raw entries (§2.2). The list-level version exists because last-write-wins is per-row and an ordering is not a row — without it, two offline reorders would silently destroy one person's work. Moving the feature to phase 2 prevents it from being deferred by association with a TBA import it never needed. |
| 2026-08-12 | 20 | **AI/MCP is setup only.** Build the two non-deferrable prerequisites (semantic field metadata, transport-agnostic use-case layer) in phase 1. No MCP endpoint, no LLM calls, no AI UI. Phases 5–6 deferred. | Keeps phase 1 small while ensuring the only expensive-to-retrofit pieces exist. Everything deferred is purely additive. |
| 2026-08-14 | 3 | **Storage = Option A (JSONB payload) with generated per-version views.** | Data volume is tiny for Postgres, so Option B's speed edge is irrelevant, while its costs (runtime DDL, per-table RLS, offline schema-mirroring, and losing multi-season history if old tables are dropped) are exactly what breaks at a venue. A gives B's query ergonomics via generated views without the risk, and is far easier to expose to an LLM (one stable shape + field dictionary). |
| 2026-08-14 | 3 | **Immutable form versioning**, but label and range edits are in-place (not a new version); range edits never invalidate existing data; keys permanent. | Preserves interpretability of historical entries while keeping the common, low-risk edits cheap during build season. |
| 2026-08-14 | 3 | **Semantic field metadata captured at creation; `description`/`unit`/`phase`/`direction` required.** | Impossible to backfill; drives generated validators/views and chart labels now, and is the LLM data dictionary later. Requiring the core four ensures they get filled in. |
| 2026-08-14 | 7 | **Offline statistics: yes — metric engine runs in the browser too.** | Ranking teams offline is the most valuable moment of an event and venue connectivity is ~zero. Feasible because the shared TS engine is the same code online and off. |
| 2026-08-14 | 1 | **Single-team install; multi-tenant out of scope.** | Simplest; retrofitting multi-tenancy touches every table and policy but isn't needed. |
| 2026-08-14 | 16 | **English LTR app chrome; Hebrew bidi-aware form content.** | Avoids full-app RTL cost while correctly rendering Hebrew labels/notes wherever they appear. Built in, not retrofitted. |
| 2026-08-14 | 15 | **TypeScript everywhere (React+Vite client, Node+Hono server, shared engine); all traffic through the server API; transport-agnostic use-case layer; runtime-generated form validation.** | One language lets the offline metric engine be written once and shared, avoiding drift on the most correctness-critical code; Zod→JSON Schema makes MCP a mechanical mapping. All-through-server is a clean single control point once cross-device realtime is deferred. |
| 2026-08-14 | 8 / 15 | **Cross-device live push deferred to nice-to-have; optimistic local UI kept.** | Removes the one thing that fought the all-through-server model and the Vercel serverless constraint; local-first UX plus refresh triggers cover the real need. |
| 2026-08-14 | 14 | **External API import (TBA/FRC) → nice-to-have.** | Wanted but not now; defers dependent scouter-assignment and result-validation features. |
| 2026-08-15 | 20 | **In-app AI panel (approach B, phase 6) → nice-to-have (§24).** MCP server (approach A, phase 5) stays parked, not nice-to-have. | Consistency: the actual LLM-in-app connection is a wanted-but-deferred feature like TBA import, so it belongs in §24. The MCP endpoint's timing is a separate future decision, so it stays parked. Setup-only prerequisites remain in phase 1. |
| 2026-08-15 | 1 | **Topic 1 CLOSED.** Team 2096, FIRST Israel district + Championship (divisions). ~11 peak users (8 scouters/2 leads/1 admin). Replaces a prior app that failed offline and across seasons. Mentor-maintained, low-maintenance multi-season, handover checklist a v1 deliverable. Soft target 2026-10-01. Success criterion + v1 non-goals confirmed. | Scale is small → confirms JSONB (§3) and all-through-server (§15) are safe. Championship participation adds division events, modelled as regular flat events (§2, Q4.3). Prior-app pain validates offline-first + year-agnostic as the primary drivers. Mentor ownership + multi-season lifespan mandates boring, documented tech and an explicit handover artifact. |
| 2026-08-17 | 2 | **Topic 2 CLOSED.** Fixed/flexible split confirmed; Event = name+year, Team = number+name. Added a **scoring model** — points entered inline per field but stored/versioned **separately** from the form version; entries hold raw observations, the score is **derived** (Q2.8d: a scoring change is never a new form version). v1 form kinds = match + super only. Scout all 6 robots incl. our own. No per-alliance data. | Fixes the stable-vs-game-specific boundary the whole app rests on. Storing raw data and deriving the score means a mid-season scoring correction re-scores history instead of fragmenting it into new form versions, and makes official-result reconciliation (Q2.5) trivial. Match+super match the team's real workflow; pit/other stay cheap to add via the `kind` field. |
| 2026-08-17 | 2 | **Official results reserved-nullable; practice/playoff stored but excluded from metrics; own robot is a regular robot.** | Reserving the results schema now avoids a later migration and costs nothing if TBA import never lands — paired with a UI rule to hide absent scores, never show blanks. Practice/playoff data is useful to browse but would distort averages, so only qualification matches feed metrics. Our robot is just one of the 6 scouted robots — no separate tracking module to build. |
| 2026-08-17 | 4 | **Championship divisions modelled as regular flat events; no division hierarchy** (Q4.3). | A division behaves like any other event for scouting; a hierarchy adds modelling cost with no scouting benefit. |
| 2026-08-17 | 4 / 16 | **App-wide active context (season + event): admin sets the default the app opens to; user overrides are session-only and never saved; only the default is cached offline. Governs both browsing and new-entry attribution; switcher on a dedicated page, not the header** (Q4.1). | A wrong active event silently ruins data, so the switch is deliberate (off the main nav) and non-sticky (always returns to the admin default). Not persisting overrides also removes per-user context storage — less data, simpler DB. |
| 2026-08-17 | 2 / 9 | **Scoring model fully specified (Q2.8a–f): non-negative points defined per field per phase; success = points > 0; scouted score = sum of field points (match/alliance total = sum across robots); no alliance-level bonus points.** | Simple, additive scoring the metric engine computes identically online/offline; success-as-points>0 avoids a second predicate to maintain; excluding penalties and alliance bonuses keeps per-robot attribution clean and matches what a scout can actually observe. |
| 2026-08-17 | 3 | **Topic 3 CLOSED.** Field catalogue = all types except Photo (deferred); Timer editable-after-stop + nullable; Event log = timestamped taps; Field-position picker stores normalized 0–1 coords on the season game image with **per-field alliance normalization** (red raw, blue mirrored H/V/both, with builder preview). List-builder + live preview UI (JSON behind advanced). **Main/active version + restorable snapshots; stats run on the main version; a metric on a missing field shows "cannot calculate this metric."** A **form belongs to a season** (several forms per season, all its events use them). Delete = **cascade behind a warning, admin-only.** Per-field `show_on_team_card`. Conditional logic kept; form-duplication dropped, JSON export/import kept. LLM metadata-suggestion → §24. | Position data is meaningless without alliance framing, so normalization is captured at the field, not backfilled. One main version keeps statistics deterministic while snapshots preserve interpretability; a loud "cannot calculate" beats silently wrong aggregates. Form-per-season matches the FRC reality (one game a year) and simplifies the event→form link. Cascade-with-warning matches the team's own preference over soft-archive. |
| 2026-08-18 | 4 | **Topic 4 CLOSED.** No event types — every event is a regular flat event (`type` dropped, Q4.2). Events weighted **equally** across a season, no event-level recency (Q4.4). **No external import in v1** — no data-import path, no `source` field on entries (Q4.5). Events model = `events(id, season_id, name, code?, is_active)`: `code` nullable and unused (reserved for deferred TBA import, §24); `start_date`/`end_date`/`location` dropped. Aggregation scopes (single event / season / all-time / custom set) adopted. | All the team's events behave identically for scouting, so a `type` field and division hierarchy add modelling cost with no benefit. Equal weighting matches how the team reasons about a season and avoids a tunable nobody would maintain. Since no data is imported, the import-only columns (`type`/dates/`location`/`source`) are dead schema — dropping them keeps the model minimal; `code` stays nullable so the deferred TBA import needs no migration. |
| 2026-08-19 | 8 | **Topic 8 CLOSED — realtime deferred.** No live/push in v1; cross-device changes propagate via the existing refresh triggers **plus a 45-second background auto-refresh** on data screens while online (Q8.1). Supabase Realtime, the live match-in-progress view (Q8.2), presence (Q8.3) and notifications (Q8.4) all move to §24 nice-to-have. **Optimistic local UI and the online/syncing/offline connection indicator are kept as core** — they are local mechanisms tied to offline (§7) and the always-visible sync indicator (§16), not cross-device push. | The team is ~11 people and venue connectivity is near zero, so realtime buys little for its cost and complexity; a 45-second refresh is trivial, predictable, and enough for a small crew. Deferring realtime also reinforces the all-through-server model (§15) by removing the one feature that argued for direct client→Supabase subscriptions. Optimistic UI and the sync indicator stay because offline-first depends on them. |
| 2026-08-19 | 7 | **Topic 7 CLOSED.** Offline scope = active competition only in IndexedDB (forms, that event's entries, stats, outbox, cached users for offline login + user-switch). Editable offline: scouting entries + admin-only pick list; not form defs / user mgmt / persisting dashboards (a lead's ephemeral draft stats page works offline). **Last-write-wins live via a base-version check; only genuine divergence (two edits branched from the same ancestor) is flagged for lead/admin "review conflicts", superseded version preserved** — normal edits and re-syncs fast-forward silently; the edit function is separate from sync (Q7.5). **Durability rule: never prune a local record before its own cloud ack** — no action (app close, handoff, QR, lead-wipe) removes an unacked record. **QR fallback ships in v1** as an animated + compressed multi-frame (fountain-coded) batch, scanned once per device-dump; it is additive & idempotent (copy by UUID, sender keeps outbox). Real workflow: central tablet gathers by QR, a runner carries it outside every few games to sync (Q7.1/Q7.2). **Full active competition cached on-device (a few MB — no photos in v1, x,y coords only) so the main statistics page and draft stats pages work offline; the main stats page is view-only offline, saving dashboards needs connectivity; active-competition-only bound; lead-code local wipe** (Q7.7). Device-to-device local-network sync → §24 nice-to-have (Q7.6). | Venue connectivity is ~zero inside the arena but exists outside, so the design is local-source-of-truth + a human-carried sync hop, with QR as the no-network path proven in FRC. UUID keys make every transfer idempotent, so more copies only increase durability; the never-prune-before-ack rule is the one guarantee that makes QR, handoffs and wipes safe. The base-version check keeps the common case (normal edits, re-syncs) friction-free — last-write-wins with no review — while catching only true two-device divergence, which is preserved for a human rather than field-merged by code no student will maintain. Because ranking teams offline is the event's most valuable moment, each device caches the full active competition's raw entries; without photos that is only a few MB, so a per-device game cap was unnecessary. A lead-gated wipe resets bad devices without risking unsynced data. |
| 2026-08-20 | 9 / 4 | **Topic 9 CLOSED.** Layer-1 metric engine ships: **menu builder, no free-text formula language** (Q9.2) with multi-field sums and a per-page display limit/target; match-subset filters (last N and/or manually excluded matches), **no recency weighting** (Q9.6); **standard deviation displayed but never a ranking sort key** while **reliability is a first-class ranking input** (Q9.5); **no cross-event percentile/z-score normalisation** (Q9.7); **minimum sample = 1**, no low-sample flag (Q9.8); **one canonical entry per (team, match)**, sync collisions resolved per §7.3 — no averaging of duplicates (Q9.4). Robot-status reliability exposes per-team breakdown/no-show/disabled counts + availability rate, surfaced in stats/ranking/pick-list. **Layer-2 official-result stats (OPR/DPR/CCWM…) → §24 nice-to-have, gated on TBA import** (Q9.3). Two entry points: a **team stat page** (that team over the active competition's matches) and an **admin-saved general stat-page builder** (leads draft-only, §5.2); **full-season slope view** orders competitions by a new admin-orderable events `sort_order` (§4.1 amendment). | Averages are what actually pick alliances, so the engine stays simple, deterministic online/off (§7.4), and honest: a dead/no-show robot never enters as a zero, duplicates never double-count, and a swingy-but-high scorer can't hide an unreliable robot because breakdown/no-show counts are first-class. A menu builder covers the team's real metrics (piece averages, success rates, `auto+teleop` sums) without the cost and safety risk of a typed evaluator. Consistency is shown for judgement but not baked into ranking per the team's call. Layer-2 stats need imported scores that don't exist in v1, so they wait with TBA. The season `sort_order` field was missing from the Q4.2 model but is required to read a metric's trend across a season left-to-right; it is display order only and does not re-weight aggregates (Q4.4 intact). |
| 2026-08-17 | 17 | **Target the Supabase free tier: store raw entries only (no persisted scores/aggregates, no photos in v1), keep DB actions simple (plain views, no materialized views/heavy triggers/cron), aggregate in the shared engine** (Q17.3). | Keeps a season comfortably within free-tier size and avoids operational complexity students can't maintain. Offline-first + per-event exports insulate scouting from the free tier's inactivity pause and caps; if limits are ever hit, plain-Postgres portability (Neon/self-host) makes migration cheap. |
| 2026-08-18 | 5 | **Topic 5 CLOSED.** Three global roles (scouter/lead/admin), no viewer/mentor, not per-event. All data visible to all roles. **Form templates admin-only; submitting entries open to all** (reaffirms Q3.1/Q3.7). Login = admin-provisioned username+password; admin-only user create (Q5.6), delete (Q5.7, preserves authored content) and promote/demote. Scouters edit own entries for 5 min from creation then the row locks — same offline & online, by client timestamp (leads/admins anytime); ~30-day offline session; switch-scouter on shared devices with per-entry attribution (Q5.8); no user audit log (Q5.5). | Small, trusted team (~11) → the cost of fine-grained per-row DB security exceeds its benefit. Roles map to the real workflow: scouters feed data, leads triage it and explore live, the admin owns structure (forms, users, saved analytics, events). Preserving a deleted user's entries protects historical data across graduating students without a separate deactivate concept. |
| 2026-08-18 | 5 / 15 / 17 | **No database-level authorization: no Postgres RLS, no per-row policies. Authorization lives in the server use-case layer and is surfaced in the UI (hide/disable).** | The team trusts its members and all traffic already flows through the single server control point (Q15.1), so the DB is not an independent attack surface to harden. One enforcement point (the use-cases) is simpler to reason about and maintain than duplicated RLS policies, and it still gates a future MCP caller since MCP reuses the same layer. Replaces the earlier RLS-on-every-table proposal and the privileged-action audit-log target in §17.1. |
| 2026-08-18 | 5 / 10 | **Ephemeral vs. persistent statistics split by role:** leads may create a **draft statistics page** (session-only, discarded on exit); creating/saving persistent dashboards is admin-only. | Lets a strategy lead explore live during a competition without polluting the shared set of saved dashboards or needing admin rights; keeps the durable analytics surface under a single owner. Full dashboard behaviour remains Topic 10. |
| 2026-08-21 | 10 | **Topic 10 CLOSED.** Full high-analysis chart set incl. pie + stacked-area/bump/bullet-gauge/small-multiples (Q10.1); **Recharts** for standard charts with **image overlays & team×metric heatmap hand-built in SVG/Canvas**, ECharts rejected as overkill (Q10.6); dashboards **shared when saved, drafts session-private** (Q10.2); scope **per-event or per-season**, built-ins on the active/selected event; phone **top-8/bottom-8** for all-teams charts, **per-team match views to 14 matches** (Q10.5); **view-time metric selector on the shared X dimension**, **expand-to-stack capped at 4** (Q10.9/Q10.10); next-year missing-field dashboards **read-only** (re-map → §24, Q10.3); **operational stats made configurable** via the same builder over app-metadata sources, not a static page (Q10.9-ops). Export (Q10.4), drill-down (Q10.7), scouted-vs-official (Q10.8, TBA-gated) → §24. | Dataset is tiny (~40 teams × ~14 matches), so ECharts' big-data engine is pure weight for a mostly-offline PWA; the only genuinely custom visuals (field/cycle overlays, heatmap-table) aren't native to either library, so Recharts + hand-built overlays gives the highest design ceiling at the lowest cost. Sharing only *saved* dashboards keeps one clean public set while leads still explore freely in private drafts. Top/bottom-N is the honest way to keep a 40-team chart readable on a phone without hiding the extremes that actually matter for picking. Making operational stats configurable (not a fixed page) reuses the whole builder and lets the team invent coverage/throughput views without a code change. |
| 2026-08-20 | 3 / 10 | **Cycle-path field type added** (amends closed Topic 3 catalogue): scouter taps an **ordered, low-fidelity list of points per cycle** (capped ≤ ~6), a list of cycles per entry, alliance-normalized like the position picker, rendered as arrowed polylines over the game image. Also confirmed **metrics are type-agnostic** — aggregations operate on any field type including **time/durations** (Timer, Event-log cycle times), offering the operations valid for each type. | Cycle routes are a core FRC strategy signal, but a precise trajectory would bloat the offline payload the team carries by QR/runner (§7); a handful of points per cycle sketches the route while staying a few KB. Stating type-agnostic metrics explicitly closes an ambiguity before Topic 10's builder is designed, so time-based metrics (cycle times) are first-class rather than a later retrofit. |
| 2026-08-21 | 11 / 10 / 4 | **Topic 11 CLOSED.** Search/browse = **dedicated pages, no global omnibox**. Canonical **basic rank = status-aware average points/match** (played matches only; no-show/disabled never count as 0) — used wherever a generic rank is needed. Five pages specified (§11.2): **team search** (season-wide, small rank badge + top-3 medals only for active-event teams, a guarded **session-only** event-switch to a team's other events that is **disabled offline** — augments the §4.1 active-context rule); **entry preview** (one entry rendered by phase, with its scouted points); **entry search** (active competition only, both match+super with a kind filter, by team/number/match/scouter, showing points); **ranking page** (admin-built metrics table, one per season no versioning; column-sort + **weighted-composite mode** — min-max 0–1 per metric, `1−norm` for `lower_is_better`, auto-normalised weights, per-team contribution breakdown, **missing metric = 0.5**, reliability a weightable column, **admin-saved presets that seed Topic 12**, offline-changeable-not-savable); **compare** (2 teams = head-to-head "final score", 3–6 = radar+table); **match preview** (six **user-entered** teams by alliance, predicted score = sum of the three teams' status-aware averages, plus reliability flags & top strengths). Q11.1 (composite ranking) = yes; **Q11.2** (notes full-text search) & **Q11.4** (TBA team-page info) → §24; **Q11.3** (multi-season history) dropped. | Dedicated pages beat one clever omnibox for a small team that thinks in "find a team / find a form". One canonical status-aware rank keeps every screen consistent and honest (a dead robot's zeros never poison an average). Weighting via simple 0–1 normalisation with a lower-is-better inversion is math a student can verify, and saving presets is exactly the pick-list seed Topic 12 needs, so the two topics compose instead of duplicating. Disabling event-switching offline respects the active-competition-only cache (§7) — the alternative silently shows an empty page. Manual team entry on match preview ships the feature now without waiting on the deferred schedule import. |
| 2026-08-21 | 13 / 10 | **Topic 13 CLOSED.** Data-quality scope kept deliberately thin for v1: **entry validation blocks only on values outside a field's declared `expected_range`** — no cross-field rules, no outlier blocking (Q13.4); **outlier/distribution flagging and the coverage matrix → §24**, the matrix gated on TBA schedule import (Q13.1); **no redundant-scouting feature** — duplicate/divergent entries ride the existing §7.3 review-conflicts path (keep last-write-wins, preserve only genuine divergence) (Q13.2); **no scouter reliability score** (Q13.3); **no dedicated bulk-fix tools** — reassign = edit the entry's team, merge = resolve a review-conflict (Q13.5); **no full per-entry edit history** (resolves the dangling Q5.5 note). Operational **meta-metric catalogue finalized** (§10.2): entries per user/match/event/total, conflicting-entry count (= review-conflicts queue), super-scouting coverage; schedule-relative coverage + sync/outbox/form-structure metrics dropped or TBA-gated. Added an **all-dashboard value-shading rule** (red→green per column, driven by `direction`; 5 domain cases incl. ordinal enums by builder order and fixed 0–100 % for rates) and a **fixed-header scrolling data-table** component (§10.1/§10.2). | Detection is worth building, but every heavy integrity feature (coverage matrix, matches-vs-scheduled, official cross-check) needs a master schedule the app won't have until TBA import lands, so v1 ships only what works schedule-free. Blocking solely on the field's own declared range is the one validation that can't produce false rejections of a weird-but-real match, and it costs nothing because `expected_range` already exists. Reusing §7.3 for conflicts and normal edits for reassignment avoids a parallel admin toolset no student would maintain. The reliability score was declined for team morale. Value-shading reuses the existing `direction` metadata (no schema change) so tables and heatmaps are legible at a glance without per-chart configuration; its edge cases (grey no-data excluded from the domain, flat mid-color when min==max, and a colorblind-safe lightness ramp with the number always shown) were pinned now so honesty and accessibility are built in, not retrofitted onto every chart. |
| 2026-08-18 | 6 | **Topic 6 CLOSED.** v1 = **manual match + team selection** (schedule-driven, **station-based** assignments → §24, gated on TBA). Form = **collapsible phase sections** (reopenable). **Sticky top match timer**: phases + countdown durations are part of the form definition; manual "Start match"; **display-only** (never gates fields, opens phases, or auto-submits). Mandatory top-level **`robot_status`** (played / no-show / disabled / broke-down-at-T); no-show & disabled hide scoring fields (no zeros). Works **portrait + landscape** on phones and tablets/iPads. All **arena-comfort** features (large targets, high-contrast/outdoor mode, large text, haptics, screen-wake-lock, thumb reach, counters). **Practice/training mode** (never stored/synced). **Undo** on counters / event-log taps / position-picker points / multi-select / timer. **Explicit submit** with confirmation summary. **Never-lose-data** local drafts. | This screen is where ~95% of usage and all data quality live, so every choice favours a fast, forgiving thumb-driven flow in a loud arena. Reopenable phases beat one-way paging because fixing a mis-tap mid-match is common. The timer is guidance, not control — hooking it to field state would fight reality (no FMS integration) and risk locking a scouter out of data entry. **`robot_status` is the headline correctness decision:** a dead/no-show robot must never record zeros (it destroys the team's average), but a robot that played then broke down did underperform for real — so those two cases diverge, and a status-aware reliability metric (Topic 9) surfaces availability without corrupting performance. |

---

## 22. Open questions index

Quick reference for what's still unanswered. Answered questions get struck through and moved into the relevant section as a confirmed requirement.

**Closed 2026-08-14:** Q1.4, Q3.1, Q3.2, Q3.9, Q7.4, Q15.1, Q15.2, Q15.4, Q15.8, Q16.2 → confirmed requirements. Q14.1 → §24 nice-to-have.
**Closed 2026-08-15:** Q1.1, Q1.2, Q1.3, Q1.5, Q1.6 → confirmed requirements (**topic 1 CLOSED**).
**Closed 2026-08-17:** Q2.1 – Q2.7 → confirmed requirements (**topic 2 CLOSED**); Q2.9 closed (own robot = regular robot); Q4.3 closed (flat events, no division hierarchy). Q2.8a–f closed (scoring model fully specified; topic 9 implements); Q4.1 closed (active context); Q17.3 closed (Supabase free tier). **Q3.3 – Q3.8 closed and Q3.10 → §24 (topic 3 CLOSED).**
**Closed 2026-08-18:** Q4.2 (no event types), Q4.4 (equal event weighting), Q4.5 (no external import / no `source` field) → confirmed requirements (**topic 4 CLOSED**). **Q5.1–Q5.8 → confirmed requirements (topic 5 CLOSED):** three global roles, form templates admin-only / entries open to all, admin-provisioned username+password, admin-only user CRUD + role changes, 5-min self-edit window (offline & online), ~30-day offline session, switch-scouter, no user audit log, **no DB RLS — authz in the use-case layer + UI**. **Q6.1–Q6.8 → confirmed requirements (topic 6 CLOSED):** manual match+team selection (assignments → §24), collapsible phases, sticky display-only match timer (phases on the form), mandatory `robot_status` + status-aware stats, portrait+landscape on phones/tablets, arena-comfort set, practice mode, undo, explicit submit, never-lose-data drafts.
**Closed 2026-08-21:** Q10.1–Q10.10 + Q10.9-ops → confirmed requirements (**topic 10 CLOSED**): full chart set (Recharts + hand-built image/heatmap overlays); dashboards shared when saved / drafts private; per-event or per-season scope, built-ins on active/selected event; phone top-8/bottom-8 for all-teams charts, per-team views to 14 matches; metric selector on shared X + expand-to-stack cap 4; configurable operational stats over app metadata. Export (Q10.4), drill-down (Q10.7), scouted-vs-official (Q10.8), next-year re-map wizard (Q10.3) → §24.

- ~~**Topic 1:** Q1.1 – Q1.6~~ ✓ **CLOSED** — all confirmed in §1.1.
- ~~**Topic 2:** Q2.1 – Q2.7~~ ✓ **CLOSED** — confirmed in §2. Own robot = regular robot (Q2.9 closed). Scoring model fully specified (Q2.8a–f closed); topic 9 implements.
- ~~**Topic 3:** Q3.1 – Q3.10~~ ✓ **CLOSED** — confirmed in §3.1. Q3.10 → §24 nice-to-have.
- ~~**Topic 4:** Q4.1 – Q4.5~~ ✓ **CLOSED** — confirmed in §4.1. Active context (Q4.1) + flat events (Q4.3) 2026-08-17; no event types (Q4.2), equal event weighting (Q4.4), no external import / no `source` field (Q4.5) 2026-08-18.
- ~~**Topic 5:** Q5.1 – Q5.8~~ ✓ **CLOSED 2026-08-18** — confirmed in §5.1–§5.2. Roles/auth/permissions settled; DB-level RLS dropped in favour of use-case-layer + UI enforcement. (Q20.6 `service`/`agent` role stays a Topic 20 dependency.)
- ~~**Topic 6:** Q6.1 – Q6.8~~ ✓ **CLOSED 2026-08-18** — confirmed in §6.2. Manual match+team selection (assignments → §24); collapsible phases; sticky display-only match timer (phases on the form); mandatory `robot_status` with status-aware stats (Topic 9); portrait+landscape on phones/tablets; full arena-comfort set; practice mode; undo; explicit submit; never-lose-data drafts.
- ~~**Topic 7:** Q7.1 – Q7.7~~ ✓ **CLOSED 2026-08-19** — confirmed in §7.1/§7.3. QR fallback in v1 (animated+compressed, Q7.1); connectivity outside arena, runner-carried tablet (Q7.2); scouting entries + admin-only pick list editable offline (Q7.3); offline stats (Q7.4, 2026-08-14); last-write-wins live via base-version check, only genuine divergence → lead/admin review (Q7.5); device-to-device local-network sync → §24 (Q7.6); full active competition cached (few MB, no photos, x,y only) for offline stats, main stats page view-only offline, lead-code wipe (Q7.7). Durability rule: never prune before cloud ack.
- ~~**Topic 8:** Q8.1 – Q8.4~~ ✓ **CLOSED 2026-08-19** — no realtime push in v1; a **45-second auto-refresh** + refresh triggers cover cross-device propagation (Q8.1). Supabase Realtime, live match-in-progress view (Q8.2), presence (Q8.3) and notifications (Q8.4) → §24 nice-to-have. Optimistic local UI and the online/syncing/offline indicator kept as core (§8.1).
- ~~**Topic 9:** Q9.1 – Q9.8~~ ✓ **CLOSED 2026-08-20** — Layer-1 metric engine (menu builder, multi-field sums, per-page limit); match-subset filters, no recency weighting; SD displayed not ranked; reliability first-class in ranking; min sample = 1; one canonical entry per (team, match) per §7.3; no cross-event normalisation. Layer-2 official-result stats → §24. Team stat page + admin-saved general stat pages (leads draft-only, §5.2); full-season slope view over an admin-orderable event `sort_order` (§4.1). Confirmed in §9.1/§9.2.
- ~~**Topic 10:** Q10.1 – Q10.10~~ ✓ **CLOSED 2026-08-21** — confirmed in §10.1/§10.2. Full high-analysis chart set (Recharts + hand-built image/heatmap overlays); dashboards shared when saved, drafts private; scope per-event or per-season, built-ins on active/selected event; phone top-8/bottom-8 for all-teams charts, per-team views to 14 matches; view-time metric selector on shared X, expand-to-stack capped at 4; **operational stats configurable via the same builder over app metadata**. Deferred to §24: export (Q10.4), drill-down (Q10.7), scouted-vs-official (Q10.8, gated on TBA), next-year field re-map wizard (Q10.3).
- ~~**Topic 11:** Q11.1 – Q11.4~~ ✓ **CLOSED 2026-08-21** — confirmed in §11.2. Dedicated search pages (no global omnibox); canonical **basic rank = status-aware avg points/match**. Five pages specified: team search (season-wide, event-only rank badges + top-3 medals, offline-disabled event switch to a team's other events), entry preview, entry search (active comp, both kinds, by team/number/match/scouter, shows points), ranking page (admin-built table + weighted-composite mode with 0–1 normalisation, `1−norm` for lower-is-better, 0.5 for missing, saved presets seeding Topic 12, reliability as a weightable column), compare (2 = head-to-head, 3–6 = radar+table), match preview (6 user-entered teams, summed-average prediction). Q11.1 composite ranking = yes; Q11.2 notes search + Q11.4 TBA team info → §24; Q11.3 multi-season history dropped.
- ~~**Topic 12:** Q12.1 – Q12.3~~ ✓ **CLOSED 2026-08-31** — confirmed in §12.2–§12.4. **In scope, moved to phase 2** (Q12.1); alliance results **entered manually**, synced now-or-next-sync (Q12.2); **admin edits, everyone else views**, the one exception being a **lead adding a do-not-pick entry with a required reason** (Q12.3). Two ordered pick lists (first/second) seeded from a §11.2 weight preset, automatic round switch, do-not-pick **blocks** an add but only **flags** a team already on a list, one-tap removal, **8 alliances × captain/pick1/pick2 + backup** with **declined** markers, derived cross-off, and a list-level version guarding reorders. Printing stays deferred (Q10.4 → §24; Q16.4 still open).
- ~~**Topic 13:** Q13.1 – Q13.5~~ ✓ **CLOSED 2026-08-21** — confirmed in §13.2. Entry validation = **hard block only on out-of-`expected_range` values** (Q13.4); outlier flagging + coverage matrix → §24 (matrix gated on TBA, Q13.1); **no redundant-scouting feature** (conflicts via §7.3 review, keep last, Q13.2); **no scouter reliability score** (Q13.3); **no dedicated bulk-fix tools** (reassign = edit team, merge = resolve conflict, Q13.5); **no full per-entry edit history**. Operational meta-metric catalogue finalized in §10.2; all-dashboard **value-shading (red→green)** color rule added to §10.2.
- **Topic 14:** Q14.1 – Q14.5
- **Topic 15:** Q15.1 – Q15.9
- **Topic 16:** Q16.1 – Q16.6
- **Topic 17:** Q17.1, Q17.2, Q17.4, Q17.5 — ~~Q17.3~~ ✓ CLOSED 2026-08-17 (Supabase free tier)
- **Topic 18:** Q18.1 – Q18.5
- **Topic 19:** Q19.1 – Q19.3
- **Topic 20:** Q20.5, Q20.6 live — Q20.1–Q20.4 and Q20.7–Q20.12 parked (LLM connection out of scope)

### Questions I'd like answered first, because they block the most work

1. ~~**Q3.1** — JSONB vs. real tables per form.~~ ✓ **CLOSED: Option A (JSONB).**
2. ~~**Q3.9** — Semantic field metadata as part of every field.~~ ✓ **CLOSED: yes; description/unit/phase/direction required.**
3. ~~**Q15.8** — The transport-agnostic use-case service layer.~~ ✓ **CLOSED: yes.**
4. ~~**Q7.4** — Must statistics work offline?~~ ✓ **CLOSED: yes — shared TS engine runs in the browser.**
5. ~~**Q16.2** — Hebrew/RTL support?~~ ✓ **CLOSED: English LTR chrome, Hebrew bidi-aware form content.**
6. ~~**Q1.4** — Single team or multi-team?~~ ✓ **CLOSED: single team.**
7. ~~**Q14.1** — External API import?~~ ✓ **CLOSED: wanted, deferred → §24 nice-to-have.**

All seven top blockers are now closed, and **topics 1–13 are fully CLOSED** — every major *feature* topic is done. Remaining: 14 (TBA import — already deferred to §24, still to be formally closed), 18 (deployment/ops), 19 (delivery phases), and the partials (15/16/17/20). The next work is **build-shaped, not feature-shaped**: close 15/16/17/18/19, then write `SPEC-FINAL.md` and `IMPLEMENTATION-PLAN.md`.

---

## 23. Change history

Every edit is logged here so changes can be audited without reprinting the document. See `COLLABORATION.md` section 6 for how to review diffs properly.

| Version | Date | Sections touched | Change |
|---|---|---|---|
| v0.24 | 2026-08-31 | 0, 0.2, 5.2, 7.3, 7.4, 8.2, 12 (rewritten & closed), 19.1, 21, 22, 23 | **Topic 12 CLOSED.** Answered Q12.1–Q12.3 and rewrote §12 into confirmed decisions (§12.2), a data model (§12.3), the three pages (§12.4) and closed questions (§12.5). In scope, **moved from phase 3 to phase 2**. **Admin-only editing, all other roles view**; a **lead may add a do-not-pick entry with a required reason** only. Two ordered pick lists (first/second) seeded from a §11.2 weight preset with automatic round switching. Do-not-pick **blocks** adding a listed team but only **flags** one already on a list; **one-tap removal** clears the flag. **Alliance bracket entered manually** — 8 × captain/pick1/pick2 + backup, **declined** markers — written locally and synced now-or-next-sync. **Cross-off derived** from the bracket; admin's device is the source of truth during selection (**resolves the §8.2 realtime dependency note**). **List-level `version`** guards whole-ordering reorders against §7.3's per-row last-write-wins. Printing stays deferred (Q10.4 → §24; Q16.4 open). Amended §5.2 (two new matrix rows), §7.3/§7.4 (offline-editable set now covers do-not-pick + bracket) and §19.1 (phases 2 and 3). |
| v0.1 | 2026-08-12 | all | Base document created from the initial requirements. 19 topics, ~100 questions. Added the fixed/flexible domain split (topic 2), venue-connectivity reality (topic 7.2), form versioning (topic 3.3), external API import (topic 14), pick list (topic 12) and data quality (topic 13) as items not in the original brief. |
| v0.2 | 2026-08-12 | 0.2, 3.2, 3.3, 3.4, 15.2, 15.3, 19.1, 22 | Added topic 20 (AI/LLM/MCP). Added semantic field metadata to 3.3 with Q3.9–Q3.10. Added transport-agnostic use-case layer to 15.2 with Q15.8–Q15.9. Noted that the MCP goal strengthens the JSONB recommendation in 3.2. Added phases 5–6. |
| v0.3 | 2026-08-12 | 0.2, 19.1, 20.1, 20.4, 21, 22 | **Scope decision:** AI/MCP is setup only — no LLM connection. Phases 5–6 marked deferred. Q20.1–Q20.4 and Q20.7–Q20.12 parked; Q20.5–Q20.6 remain live. First Decision Log entry added. |
| v0.4 | 2026-08-12 | 23 | Added this change history section. Companion file `COLLABORATION.md` created (process rules, answer shorthand, session templates, review workflow, build-phase rules). |
| v0.5 | 2026-08-12 | — | No spec changes. `COLLABORATION.md` v1.1 adds section 10 (working in Claude Code: surface choice, `CLAUDE.md`, `/clear` discipline, plan mode, review loop, permissions). `CLAUDE.md` created for the repo root. |
| v0.6 | 2026-08-14 | 0, 1.1, 3.1, 7.1, 8.1, 15.1, 16.1, 21, 22, 24 (new) | Closed the seven top blockers plus Q3.2/Q15.1/Q15.2/Q15.4: storage = Option A (JSONB); immutable versioning (label/range edits in place); semantic metadata required (desc/unit/phase/direction); offline stats yes; single-team; English LTR chrome + Hebrew bidi form content; all-traffic-through-server; TypeScript everywhere (React+Vite / Node+Hono / shared engine); runtime-generated form validation. Cross-device realtime and TBA/FRC import moved to new §24 nice-to-have. Decision Log rows added. `nice` shorthand added to `COLLABORATION.md` §3 and `CLAUDE.md`. |
| v0.7 | 2026-08-15 | 0.2, 1.3, 3.4, 7.4, 14.3, 15.3, 16.3, 19.1, 20.1, 21, 24 | **Consistency pass.** Struck through the already-answered questions still printed as open in their per-topic lists (Q1.4, Q3.1/Q3.2/Q3.9, Q7.4, Q14.1, Q15.1/Q15.2/Q15.4/Q15.8, Q16.2) with pointers to where each is confirmed. Set topics 1/3/7/8/15/16 to PARTIAL in the §0.2 status table. Moved the in-app AI panel (phase 6) to §24 nice-to-have; MCP server (phase 5) stays parked. Decision Log row added. |
| v0.8 | 2026-08-15 | 0, 0.2, 1.1, 1.2, 1.3, 2.3, 21 | **Topic 1 CLOSED.** Answered Q1.1/Q1.2/Q1.3/Q1.5/Q1.6; moved §1.2 success criterion + non-goals to confirmed. Added confirmed facts: team 2096, Israel district + Championship (divisions), ~11 peak users, prior-app pain points, mentor low-maintenance lifespan + handover-checklist deliverable, soft 2026-10-01 target. Flagged the 8-scouters-vs-6-stations coverage question on Q2.3. Status table topic 1 → CLOSED. Decision Log row added. |
| v0.9 | 2026-08-17 | 0.2, 1.1, 2.1, 2.2, 2.3, 3.3, 4.3, 21, 22 | **Topic 2 CLOSED.** Answered Q2.1–Q2.9. Event = name+year, Team = number+name. Scoring model confirmed as **points entered inline per field but stored in a separately-versioned model; entries hold raw data, score is derived; a scoring change is never a new form version** (Q2.8d closed; Q2.8a–c/e/f spun out to topics 3/9). Own robot is a **regular robot** — no separate log (Q2.9). Form kinds cut to match+super; scout all 6 robots incl. ours; no per-alliance data; official results reserved-nullable with a no-empty-box UI rule; practice/playoff excluded from metrics. Also closed **Q4.3** — championship divisions are regular flat events, no hierarchy (topic 4 → PARTIAL; §1.1 rationale amended). Added a forward-reference in §3.3. Decision Log + status table updated. |
| v0.10 | 2026-08-17 | 0.2, 4.1, 4.3, 16.1, 17.1, 17.2, 21, 22 | Added the **app-wide active context** — admin sets the default the app opens to; user overrides are session-only (not saved); only the default is cached offline; governs browsing + new-entry attribution; switcher on a dedicated page, not the header. Closed **Q4.1**, cross-noted in §16.1. Adopted the **Supabase free tier** as an explicit constraint — raw-only storage, simple DB actions, aggregate in the shared engine; closed **Q17.3** with pause/cap mitigations and a plain-Postgres fallback. Status table + Decision Log updated. |
| v0.11 | 2026-08-17 | 2.1, 2.2, 2.3, 3.3, 21, 22 | **Scoring model fully closed (Q2.8a–f).** Non-negative points defined per field per phase; success = points > 0; scouted score = sum of field points (match/alliance total = sum across robots); no alliance-level bonus points. Updated glossary + §2.2, struck the Q2.8 sub-questions, refreshed the §3.3 forward-reference; Decision Log row added. |
| v0.13 | 2026-08-18 | 0, 0.2, 4.1, 4.2, 21, 22, 23 | **Topic 4 CLOSED.** Answered Q4.2/Q4.4/Q4.5. No event types — every event is a regular flat event (`type` dropped). Events weighted **equally** across a season (no event-level recency). **No external import in v1** — no data-import path, no `source` field on entries. Events model trimmed to `events(id, season_id, name, code?, is_active)`: `code` nullable/unused (reserved for the deferred TBA import, §24); `start_date`/`end_date`/`location` removed. Aggregation scopes adopted. Folded §4.2 proposals into §4.1 confirmed; renumbered old §4.3 → §4.2. Status table + Decision Log + open-questions index updated. |
| v0.14 | 2026-08-18 | 0, 0.2, 5 (rewritten), 10.2, 10.3, 17.1, 21, 22 | **Topic 5 CLOSED.** Answered Q5.1–Q5.8. Three global roles (scouter/lead/admin), no viewer/mentor, not per-event; confirmed permission matrix (§5.2). Form templates admin-only, entries open to all (reaffirms Q3.1/Q3.7). Admin-provisioned username+password; admin-only user create/delete/promote-demote; deletion preserves authored content. Scouters self-edit own entries for 5 min from creation, then the row locks — same offline & online, ~30-day offline session (decided-by-Claude); switch-scouter + per-entry attribution; no user audit log. **Dropped DB-level RLS/per-row policies and the privileged-action audit-log target — authorization now lives in the server use-case layer + UI (§17.1 updated).** Lead-only ephemeral "draft statistics page" vs. admin-only saved dashboards. Decision Log (4 rows) + status table + open-questions index updated. Also recorded in Topic 10 the requested **static general statistics page** (operational/meta stats) alongside the dynamic robot dashboards, with candidate contents and new Q10.9. |
| v0.15 | 2026-08-18 | 0, 0.2, 6 (rewritten), 9.2, 21, 22, 24 | **Topic 6 CLOSED.** Answered Q6.1–Q6.8. v1 manual match+team selection; schedule-driven **station-based** assignments → §24 (gated on TBA). Collapsible, reopenable phase sections. **Sticky top match timer** whose phases + countdown durations live on the form definition; manual "Start match"; **display-only** (never gates fields/opens phases/auto-submits). Mandatory top-level **`robot_status`** (played / no-show / disabled / broke-down-at-T); no-show & disabled hide scoring fields (no zeros); **status-aware statistics** — performance metrics count played+broke-down, exclude no-show+disabled, plus a reliability/availability metric (added to §9.2). Portrait+landscape on phones & tablets/iPads. Full arena-comfort set (large targets, high-contrast/outdoor, large text, haptics, screen-wake-lock, thumb reach, counters). Practice/training mode (never stored/synced). Undo on counters/event-log/position-picker/multi-select/timer. Explicit submit with confirmation summary. Never-lose-data local drafts. Status table, Decision Log, open-questions index, §24 updated. |
| v0.17 | 2026-08-19 | 0, 0.2, 8 (rewritten), 15.2, 19.1, 21, 22, 24 | **Topic 8 CLOSED — realtime deferred.** Answered Q8.1–Q8.4. No realtime/live push in v1; cross-device changes propagate via refresh triggers **+ a 45-second background auto-refresh** while online (Q8.1). Supabase Realtime, live match-in-progress view (Q8.2), presence (Q8.3), notifications (Q8.4) → §24 nice-to-have (four rows). **Kept as core:** optimistic local UI and the online/syncing/offline connection indicator (local, tied to §7/§16 — not cross-device push). Removed "live views and presence" from §19.1 phase 3. **Also cleaned §15.2:** replaced the stale client↔Supabase "hybrid" proposal (which still recommended direct realtime subscriptions) with the confirmed **all-through-server** model (Q15.1), noting the realtime path is the deferred §24 item; optimistic UI retained. Status table, Decision Log, open-questions index, change history updated. |
| v0.16 | 2026-08-19 | 0, 0.2, 7.3, 7.4, 19.1, 21, 22, 24 | **Topic 7 CLOSED.** Answered Q7.1–Q7.3, Q7.5–Q7.7. Offline store scoped to the active competition only (+ cached users for offline login/switch). Editable offline: scouting entries + admin-only pick list. **Last-write-wins live via a base-version check; only genuine divergence goes to lead/admin "review conflicts" (superseded version preserved); the edit function is separate from sync** (Q7.5). **Durability rule: never prune a local record before its own cloud ack** (no action removes an unacked record). **QR fallback promoted to phase 1** as an animated + compressed, fountain-coded multi-frame batch scanned once per device-dump; additive & idempotent by UUID (sender keeps outbox). Workflow: central tablet gathers by QR → runner syncs outside every few games (Q7.1/Q7.2). **Full active competition cached on-device (a few MB, no photos in v1 — x,y only) so offline stats work; main stats page view-only offline, draft stats creatable offline; active-competition-only bound; lead-code local wipe** (Q7.7). Device-to-device local-network sync → §24 (Q7.6). Status table, Decision Log, open-questions index, §19.1 roadmap, §24 updated. |
| v0.12 | 2026-08-17 | 0.2, 1.1, 3.1, 3.3, 3.4, 4.1, 21, 22, 24 | **Topic 3 CLOSED.** Answered Q3.3–Q3.8; Q3.10 → §24. Field catalogue = all types except Photo (deferred); Timer editable-after-stop + nullable; Event log = timestamped taps; Field-position picker stores normalized 0–1 coords on the season game image with **per-field alliance normalization** (red raw / blue mirrored H/V/both, with preview). List-builder + live-preview UI (JSON behind advanced). **Main/active version + restorable snapshots; stats on the main version; missing-field metric → "cannot calculate this metric."** Form belongs to a **season** (several forms per season) — reconciled §1.1 and §4.1 wording from competition→form to season→form. Delete = **cascade behind a warning, admin-only.** Per-field `show_on_team_card`. **Removed the per-field offline flag — every field is scoutable offline.** Conditional logic kept; form-duplication dropped, JSON export/import kept. Status table + Decision Log updated; LLM metadata-suggestion added to §24. |
| v0.20 | 2026-08-21 | 0.2, 10 (closed), 21, 22, 23, 24 | **Topic 10 CLOSED.** Answered Q10.1–Q10.10 + the operational-stats question. Full chart set (Recharts + hand-built SVG/Canvas image & heatmap overlays); dashboards shared when saved / drafts private; per-event-or-season scope with built-ins on the active/selected event; phone top-8/bottom-8 for all-teams charts, per-team views to 14 matches; view-time metric selector on the shared X + expand-to-stack capped at 4; next-year missing-field dashboards read-only; **operational (meta) statistics made configurable via the same builder over app-metadata sources** (was a static page). Moved to §24: export (Q10.4), drill-down (Q10.7), scouted-vs-official (Q10.8, TBA-gated), next-year field re-map wizard (Q10.3). Status table, Decision Log, open-questions index, §24 updated. |
| v0.21 | 2026-08-21 | 0.2, 5.1, 5.3, 10.1, 10.2, 13 (closed), 21, 22, 23 | **Topic 13 CLOSED.** Answered Q13.1–Q13.5. Entry validation = **hard block only on out-of-`expected_range` values**; outlier flagging + coverage matrix → §24 (matrix TBA-gated); **no redundant-scouting feature** (conflicts via §7.3 review, keep last); **no scouter reliability score**; **no dedicated bulk-fix tools** (reassign = edit team, merge = resolve conflict); **no full per-entry edit history** (resolved the dangling Q5.5 cross-reference). Rewrote §13.2/§13.3 (proposed → confirmed + closed questions). In §10.2: **finalized the operational meta-metric catalogue** (entries per user/match/event/total, conflicting-entry count, super-scouting coverage; rest dropped or TBA-gated) and added the **all-dashboard value-shading rule** (red→green per column via `direction`; 5 domain cases; domain follows dashboard scope/filters) plus its **edge cases** — grey "—" for no-data (excluded from domain), flat mid-color when min==max, and colorblind safety via a monotonic lightness ramp + always showing the number. In §10.1: **data table** upgraded to a fixed-height scroll viewport with frozen header + sticky first column and value shading. Status table, Decision Log, open-questions index, summary line updated. |
| v0.23 | 2026-08-21 | 0.2, 10.2, 11 (closed), 21, 22, 23, 24 | **Topic 11 CLOSED.** Answered Q11.1–Q11.4. Rewrote §11.2/§11.3 into five specified pages (team search, entry preview, entry search, ranking, compare, match preview); dropped the global omnibox; defined the **canonical status-aware basic rank** and the **weighted-composite ranking** (0–1 normalise, `1−norm` for lower-is-better, 0.5 for missing, saved presets seeding Topic 12, reliability weightable); guarded **offline-disabled event switch** from team search (augments §4.1). Filled the §10.2 ranking/compare/match-preview built-in stubs with cross-refs to §11.2 and marked the coverage/quality built-in TBA-gated. Q11.2 (notes search) + Q11.4 (TBA team info) → §24; Q11.3 (multi-season history) dropped. Status table, Decision Log, open-questions index, summary line updated. |
| v0.22 | 2026-08-21 | 20.3, 20.4, 23 | Added a **usage/help agent** as a distinct, lighter third AI surface in §20.3 (documentation Q&A about operating the app; no MCP, no data access, no RAG — just an LLM connection + a maintained help.md; internet-only, build-season aid; ~1–2 days once an LLM connection exists). **Parked** with new questions Q20.13 (want it / priority vs. the data panel), Q20.14 (help-corpus ownership), Q20.15 (shared vs. separate cheaper model). No decision made — recorded like the parked MCP/AI items. |
| v0.19 | 2026-08-20 | 3.1, 10.2, 10.3, 21, 23 | **Dashboards working session (Topic 10 still OPEN).** Folded into §10.2: metric definitions are season-level/event-agnostic with **scope bound per-dashboard** (season-or-subset *or* one event, draft & saved alike); a **12-column responsive grid** with per-tile width span + drag reorder (two stat cards per row); a **view-time metric selector** + **expand-to-stack**; standard pages reframed as **built-in dashboards** plus a **General Dashboards hub** listing all season dashboards and prompting for input when opened without page context. Added **Cycle-path field type** to §3.1 (ordered low-fidelity points per cycle, ≤ ~6, alliance-normalized, arrowed-polyline overlay) and a matching **cycle-path overlay** chart type + explicit **type-agnostic metrics** note (incl. time/durations) in §10.2 — both settled (Decision Log row). Annotated Q10.2 as largely resolved; added Q10.9 (metric-family derivation) and Q10.10 (expand-stack cap). |
| v0.18 | 2026-08-20 | 0, 0.2, 4.1, 9 (rewritten), 21, 22, 24 | **Topic 9 CLOSED.** Answered Q9.1–Q9.8. Layer-1 engine = **menu builder, no formula language** (multi-field sums + per-page display limit); match-subset filters (last N / manual exclude), **no recency weighting**; **SD displayed, not ranked**; **reliability first-class in ranking** with per-team breakdown/no-show/disabled counts + availability rate surfaced in stats/ranking/pick-list; **no cross-event normalisation**; **min sample = 1**; **one canonical entry per (team, match)**, sync collisions per §7.3 (no duplicate averaging). **Layer-2 official-result stats → §24, gated on TBA** (Q9.3). Added **team stat page** + **admin-saved general stat-page builder** (leads draft-only, §5.2) and a **full-season slope view**. §4.1 amendment: new admin-orderable events **`sort_order`** (display order only, doesn't re-weight — Q4.4 intact) **[RAISED BY ME]**. Fixed stale "if Q7.4" conditional in §9.2. Status table, Decision Log, open-questions index, §24 updated. |

---

## 24. Nice-to-have (wanted, deferred)

Confirmed as **wanted but deliberately out of current scope** — revisit later. Distinct from *parked* (topic 20 — timing undecided pending a separate decision) and *skip* (out of scope permanently). Shorthand `Q_ nice` moves an item here.

| Item | Source | Note |
|---|---|---|
| **External data import (TBA / FRC Events API)** — event, team, schedule and result import | Q14.1, topic 14 | Wanted, not now. Deferring it also defers schedule-driven **scouter assignments** (Q6.1/Q6.2) and **official-result validation** (topic 13), which depend on it. |
| **Layer-2 derived FRC statistics** — OPR, DPR, CCWM, win rate, avg ranking points, schedule strength, computed from official match scores | Q9.3, topic 9 | Wanted, not now. A cross-check on scouting data; depends on the TBA import above. v1 ships Layer-1 metrics (from scouting) only. |
| **Schedule-driven scouter assignments** — auto-fill "you are watching Red 2, team 1577" per match | Q6.1/Q6.2, topic 6 | Wanted, not now. When built it is **station-based** (fixed station → fixed alliance slot). Depends on the imported match schedule above. v1 uses manual match + team selection. |
| **Cross-device live updates (Supabase Realtime)** — another device's changes appearing without a refresh | Q15.1, Q8.1, topics 8 & 15 | Optimistic local UI, refresh-on-triggers, and a **45-second auto-refresh** ship now (§8.1); automatic cross-device push is deferred. Would reintroduce a direct client→Supabase subscription path (the rejected §15.2 hybrid); additive later, but note it partially reopens the all-through-server model. |
| **Live "match in progress" view** — the strategy lead watching the six scouters' entries fill in during a match | Q8.2, topic 8 | Wanted, not now. Requires connectivity usually absent inside a venue. |
| **Presence** — who is online, which scouters have/haven't submitted for the current match | Q8.3, topic 8 | Wanted, not now. Catches "station 4 hasn't submitted for 3 matches" during the event; depends on live connectivity. |
| **Notifications** — "next match in 4 min", "you have unsynced data", "conflict needs review" | Q8.4, topic 8 | Wanted, not now. In-app vs. push to be decided when built. |
| **Device-to-device local-network sync** — one device runs a tiny local server on a shared WiFi/hotspot (no internet, no cables) and others sync to it | Q7.6, topic 7 | Wanted, not now. Moves a whole event's data at once, faster than QR, but needs all devices on the same local network. v1 uses QR fallback + a runner carrying one tablet outside to sync. |
| **In-app AI insights panel** — server-side LLM orchestration, in-app chat/insights UI, cached briefings, notes summarisation, NL-to-chart | Topic 20 (approach B), phase 6 | Wanted, not now. Requires internet so unavailable at most of a competition; generated insights would be cached for offline reading. The setup-only prerequisites (semantic field metadata §3, use-case layer §15) stay in phase 1. **MCP server (phase 5) is *not* here — it stays parked in topic 20.** |
| **LLM-suggested field metadata** — paste the game manual's scoring section, get proposed fields with descriptions/units/ranges/directions to edit | Q3.10, topic 3 | Wanted, not now. Turns building a new season's form from an evening into minutes; cheapest payoff of the AI work. Depends on the same LLM connection deferred in topic 20. |
| **Dashboard/chart export** — chart as PNG, table as CSV/Excel, dashboard as PDF | Q10.4, topic 10 | Wanted, not now. Printed pick lists and match previews are common in pits; v1 is screen-only. |
| **Chart drill-down / click-through** — click a bar → that team's page, click a line point → that match's raw entry | Q10.7, topic 10 | Wanted, not now. Turns charts into a navigation surface; v1 charts are read-only visuals. |
| **Scouted-vs-official comparison** — scouted contribution vs OPR and other official figures, to validate scouting quality | Q10.8, topic 10 | Wanted, not now. Depends on the TBA/FRC import and Layer-2 stats above. |
| **Next-year dashboard re-map wizard** — re-point a saved dashboard's fields onto a new season's form | Q10.3, topic 10 | Wanted, not now. v1 renders such dashboards read-only and flags missing fields; the guided re-map is the deferred convenience. |
| **Full-text notes search** — search free-text scouter comments ("who was mentioned as having a fast climb?") | Q11.2, topic 11 | Wanted, not now. Needs a Postgres text index and a decision on how it works offline. v1 entry search covers team/number/match/scouter. |
| **TBA public info on the team page** — awards, event history, robot photos, external links | Q11.4, topic 11 | Wanted, not now. Cheap once TBA import lands; gated on that import (§14 → §24). |
