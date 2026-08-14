# FRC Scouting Platform — Design Specification (Living Document)

**Status:** DRAFT v0.6 — core architecture forks decided (storage, versioning, offline stats, stack, access model, language). Topics remain OPEN pending their sub-questions.
**Last updated:** 2026-08-14
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
| 1 | Product vision & scope | OPEN |
| 2 | FRC domain model & glossary | OPEN |
| 3 | Dynamic forms — the core architecture decision | OPEN |
| 4 | Seasons, competitions & events | OPEN |
| 5 | Users, roles, authentication & permissions | OPEN |
| 6 | Scouting data entry (runtime UX) | OPEN |
| 7 | Offline-first & synchronisation | OPEN |
| 8 | Realtime (live updates, no refresh) | OPEN |
| 9 | Statistics engine & computed metrics | OPEN |
| 10 | Dynamic dashboards, graphs & visualisations | OPEN |
| 11 | Search, ranking & browse | OPEN |
| 12 | Alliance selection & pick list | OPEN |
| 13 | Data quality, integrity & scouter reliability | OPEN |
| 14 | External data integration (TBA / FRC Events API) | OPEN |
| 15 | System architecture, repo layout & services | OPEN |
| 16 | UI/UX design system & responsive behaviour | OPEN |
| 17 | Non-functional requirements | OPEN |
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
- Data is **organised by competition**, and each competition is linked to the form that represents the game being played.
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

### 1.2 Proposed decisions

**Primary success criterion.** I suggest we define one, because it decides a lot of trade-offs: *"During a competition, six scouters can each record one robot per match on a phone with no internet, and within 60 seconds of regaining connectivity the strategy lead sees updated team rankings on a laptop."* If that works, the product works.

**Explicit non-goals for v1** (to keep scope sane — challenge any of these):

- Not a public/multi-tenant SaaS for other teams (single-team install, though see Q1.4).
- No native iOS/Android app — a PWA installable to the home screen.
- No live-video or camera-based automatic scouting.
- No integration with the FMS/field systems.

### 1.3 Open questions

- **Q1.1** — How many people use this at a competition, roughly? (scouters, strategy leads, admins) This drives realtime volume, sync conflict likelihood, and how much the UI needs to guard against mistakes.
- **Q1.2** — What is your **team number and district/region**? You're in Israel — is this the FRC Israel district, and do you attend Israeli district events plus possibly a championship? This affects the event/season model and the external API integration.
- **Q1.3** — What are you using today (paper, Google Sheets, another app)? What specifically fails about it? The pain points are the highest-value requirements.
- **Q1.4** **[RAISED BY ME]** — Should the system support **multiple teams/organisations** in one deployment (multi-tenant), or is it strictly your team? Building single-tenant is much simpler; retrofitting multi-tenancy later is expensive because it touches every table and every security policy. Decide now even if the answer is "single team".
- **Q1.5** — Is there a hard deadline (e.g. ready before kickoff in January, or before a specific event)? This determines the phasing in topic 19.
- **Q1.6** **[RAISED BY ME]** — Who maintains this after you? If you're graduating, the spec should favour boring, well-documented technology over clever solutions.

---

## 2. FRC domain model & glossary

### 2.1 Why this section exists **[RAISED BY ME]**

You said "year-agnostic", which is right — but there is a set of concepts that is *stable across every FRC season*. Modelling those as first-class entities (rather than as fields inside a dynamic form) is what makes the app genuinely reusable. If team number or match number lives inside the flexible JSON blob, then ranking, search and cross-season comparison all break. So we split the world in two:

**Fixed skeleton (hard-coded, same every year):**

| Entity | Meaning |
|---|---|
| Season | A year, e.g. 2026. Has one game. |
| Event / Competition | A specific competition (district event, regional, championship division, or an internal scrimmage). Belongs to a season. |
| Team | An FRC team: number, name, rookie year, region. Global, spans seasons. |
| Match | A match at an event: type (practice / qualification / playoff), match number, the 3 red teams, the 3 blue teams, optionally the official result. |
| Scouter (User) | A person entering data. |
| Scouting Entry | One observation: one scouter, one team, one match, one form version → a payload of answers. |

**Flexible layer (created by you in the app, differs every year):**

| Entity | Meaning |
|---|---|
| Form | The game-specific questionnaire ("2026 match scouting"). |
| Form Version | An immutable snapshot of a form's fields. |
| Field | One question: type, label, validation, section, conditional logic. |
| Metric | A computed value derived from fields (e.g. "total game pieces = auto + teleop"). |
| Dashboard / Chart | A saved visualisation configuration referencing fields and metrics. |

### 2.2 Proposed decisions

- **Team, Event, Match, Season are fixed schema.** Every scouting entry carries structured foreign keys to these, plus a flexible payload for the game-specific answers.
- **Team numbers are global and stable** — team 1577 in 2024 and 2026 is the same team, so multi-season team history works for free.
- **Multiple form types per season.** Not everything is match scouting. I strongly suggest supporting at least these form *kinds*, because they have different cardinality and you will want them:

| Form kind | One record per | Purpose |
|---|---|---|
| Match scouting | (team, match) | Quantitative performance in a match. |
| Pit scouting | (team, event) | Robot specs: drivetrain, weight, mechanisms, language, photos. |
| Super scouting / qualitative | (team, match) or (team, event) | Driver skill, defence, penalties, breakdowns — the subjective read. |
| Human/other | free | Anything else you invent later. |

### 2.3 Open questions

- **Q2.1** — Do you agree with the fixed/flexible split, or do you want *everything* (including team and match number) to be a configurable field? (I recommend the split — strongly. Ask me why if you want the full argument.)
- **Q2.2** — Do you want the multiple **form kinds** above in v1, or only match scouting first with the others later?
- **Q2.3** **[RAISED BY ME]** — Do you scout **all 6 robots in a match** (yours plus opponents), or only opponents/partners? Do you scout your *own* robot too? This changes scouter assignment and coverage reporting.
- **Q2.4** **[RAISED BY ME]** — Do you need to record data **per alliance** (e.g. total alliance score, penalties on the alliance) as opposed to per robot? Some things genuinely aren't attributable to one robot.
- **Q2.5** **[RAISED BY ME]** — Do you want to store **official match results** (scores, ranking points, win/loss) alongside your scouted data? It's very useful for validation ("our scouted total says 42, official says 51 — someone missed something") and it comes free from the API in topic 14.
- **Q2.6** — Do you need **practice matches** and **internal scrimmages** (your own team's testing days) as first-class events, or only official competitions?
- **Q2.7** **[RAISED BY ME]** — Should the app track **your own team's robot** across the season (reliability log, mechanism changes, match notes from the drive team)? It's adjacent but often wanted.

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
| Timer / stopwatch | Accumulate time (e.g. time spent defending). |
| Event log | Timestamped taps — enables cycle-time analysis. |
| Field position picker | Tap on an image of the field, stores x/y. Great for shot-location heatmaps. |
| Photo | Robot pictures, pit scouting. Needs Supabase Storage + offline queueing. |
| Computed | Read-only, derived from other fields by a formula. |
| Section / header | Layout only, no data. |

**Field configuration per field:** key, label, help text, type, required, default value, min/max, options list, section/page, display order, visibility condition, whether it appears in the quick summary, whether it is scoutable offline.

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

**Conditional logic:** simple rules only — `show this field if <field> <op> <value>`. Not a general expression language.

**Form templates:** you can duplicate last year's form as a starting point, and export/import a form definition as JSON so it can be shared or version-controlled.

### 3.4 Open questions

- **Q3.1** — Do you accept **Option A (JSONB)**? If you have a strong reason to want real tables per form, tell me and I'll re-argue it.
- **Q3.2** — Do you accept **immutable form versioning**, including "you cannot silently change a form that already has data"? The alternative is simpler to build but will eventually corrupt your historical analysis.
- **Q3.3** — Which field types from the catalogue above do you actually want, and is anything missing? Specifically: do you want **field-position picking** (tap the field diagram) and **event logs with timestamps** (for cycle times)? These are high value in FRC and non-trivial to build.
- **Q3.4** — How is the form **built** in the UI? (a) a list-based builder — add field, pick type, configure, reorder by drag; (b) a visual WYSIWYG canvas with live preview; (c) raw JSON editor for power users, with a UI on top. I'd suggest (a) + live preview pane, plus (c) hidden behind an "advanced" toggle.
- **Q3.5** — When you aggregate across an entire season and a form changed mid-season, what should happen to a metric whose field only exists in some versions? (a) exclude entries missing the field; (b) treat as zero; (c) refuse and warn the user; (d) require an explicit field-mapping when creating the metric.
- **Q3.6** **[RAISED BY ME]** — Can the **same form be reused across several competitions** in a season (I assume yes — same game), and can **one competition have several forms** (match + pit + super)? I'm assuming a many-to-many link table. Confirm.
- **Q3.7** **[RAISED BY ME]** — What does "**delete a form**" mean when it has thousands of entries? Options: block deletion, soft-archive (hidden but data preserved), or cascade delete behind a scary confirmation. I recommend archive-by-default with hard delete only for empty forms.
- **Q3.8** **[RAISED BY ME]** — Do you want a per-field **"show on the live team card"** flag, so the strategy lead's quick view is configurable per year rather than hard-coded?
- **Q3.9** **[RAISED BY ME]** — Do you accept the **semantic metadata** block above as part of every field? Which of those attributes should be **required** when creating a field, and which optional? (My suggestion: description, unit, phase and direction required; the rest optional. Required fields are annoying at 2am during build season, but optional metadata never gets filled in.)
- **Q3.10** **[RAISED BY ME]** — Should the form builder be able to **suggest** this metadata with an LLM? You paste the game manual's scoring section, and it proposes fields with descriptions, units, ranges and directions, which you then edit. This makes creating a new season's form take minutes instead of an evening — and it's the cheapest possible payoff from the AI work. Interested?

---

## 4. Seasons, competitions & events

### 4.1 Confirmed requirements

- Data is organised by competition name.
- Each competition is linked to the form representing that game.
- Search/filter data by competition, and by all competitions within one year.

### 4.2 Proposed decisions

- `seasons(year, game_name, field_image_url)` → `events(id, season_id, name, code, type, start_date, end_date, location, is_active)`.
- An **"active event"** concept: the app knows which event is happening now so scouters land directly on the right data-entry screen with the event pre-selected. Getting the wrong event selected is a classic source of ruined data.
- Event `code` maps to the official event key (e.g. TBA's `2026isde1`) to enable schedule import in topic 14.
- Aggregation scopes available everywhere: **single event**, **season (all events)**, **multi-season / all time**, and **custom set of events**.

### 4.3 Open questions

- **Q4.1** — Do you want a "**current event**" that admins set globally, or should each scouter pick their own event each session? (Global with an override is safer.)
- **Q4.2** — Event types you need: official district event, regional, championship (with divisions), off-season event, internal scrimmage, practice day. Which of these matter?
- **Q4.3** **[RAISED BY ME]** — Championship events have **divisions** (a team plays in one division, then possibly the field of champions). Do you need to model that, or is a flat event list enough for now?
- **Q4.4** — When comparing across a season, should events be **weighted** or treated equally? (E.g. later events reflect an improved robot — do you want recency weighting in stats? See also Q9.6.)
- **Q4.5** **[RAISED BY ME]** — Do you need to scout **teams you will never play** (e.g. importing another team's shared data, or scouting from a stream before an event)? That implies a data-import path and a "source" field on entries.

---

## 5. Users, roles, authentication & permissions

### 5.1 Confirmed requirements

- Create users for scouters.
- An admin user who can perform more complex actions on the database.

### 5.2 Proposed decisions

**Roles (draft):**

| Role | Can |
|---|---|
| Scouter | Log in, see their assignment, submit and edit **their own** entries (within a time window), view basic team pages. |
| Lead / Strategist | Everything a scouter can, plus view and edit all entries, build dashboards, manage the pick list, see data-quality reports. |
| Admin | Everything, plus create/edit/delete forms, manage users and roles, manage events, import external data, export/delete data. |
| Viewer (optional) | Read-only — mentors, other students, alliance partners. |

**Authentication:** Supabase Auth. The important sub-question is *how* people log in, because at a competition, 15-year-olds on borrowed phones with no internet must be able to start scouting in under 10 seconds.

- Option 1: email + password (Supabase default). Reliable, but slow to type and kids forget passwords.
- Option 2: admin-provisioned username + short PIN. Fastest, and works offline if the session is cached. Requires a custom flow on top of Supabase Auth.
- Option 3: magic link / OTP by email — requires internet, so unusable at a venue. Not viable alone.
- My suggestion: email+password for leads/admins, plus **long-lived cached sessions** for scouter devices so a scouter authenticates once at the hotel and stays logged in all weekend offline.

**Authorisation:** Supabase **Row Level Security** on every table, with role stored in a `profiles` table and read via a JWT claim. Privileged operations (form creation, user management, bulk delete, imports) go through the server service using the service-role key, never from the browser.

### 5.3 Open questions

- **Q5.1** — Are the roles above right? Do you need a **mentor/viewer** role, or a **per-event** role (someone is a lead at one event and a scouter at another)?
- **Q5.2** — How do scouters log in? Which option in 5.2, or something else?
- **Q5.3** — Can a scouter **edit or delete their own entry** after submitting? Within a time limit? Only before it syncs? (Recommendation: edit freely until the match is confirmed by a lead; full history kept.)
- **Q5.4** — Who can see what? Is all scouting data visible to all scouters, or should a scouter only see their own submissions? (Some teams hide aggregate data from scouters to prevent bias — "I'll just enter what the average says".)
- **Q5.5** **[RAISED BY ME]** — Do you need an **audit log** (who changed what, when, from what to what)? For a system where teenagers can edit competition-critical data, I think yes, at least for entry edits, deletions, and form changes.
- **Q5.6** **[RAISED BY ME]** — Self-registration or admin-created accounts only? Admin-only is safer; you can invite by link.
- **Q5.7** **[RAISED BY ME]** — Team members graduate every year. Do you need **deactivate user** (preserving their historical entries) as distinct from delete?
- **Q5.8** **[RAISED BY ME]** — Shared devices: if the team owns 6 tablets used by different scouters each shift, do you want a "**switch scouter**" quick action rather than full logout/login? And should the *entry* record who actually entered it even on a shared device?

---

## 6. Scouting data entry (runtime UX)

### 6.1 Why this matters more than it sounds **[RAISED BY ME]**

This is where 95% of the app's actual usage happens: a student standing in a loud arena, phone in one hand, watching a robot move fast, with 2 minutes and 30 seconds of match plus ~10 seconds between matches. If this screen is even slightly awkward, the data quality collapses and every graph downstream is worthless. I'd like to spend real time on this topic.

### 6.2 Proposed decisions

- **Assignment-driven flow.** The scouter opens the app and sees: *"Match 34 — you are watching Red 2, team 1577."* One tap to start. No hunting through dropdowns. This requires a match schedule (topic 14) and a scouter assignment feature.
- **Phase-based layout.** The form is split into pages matching match phases (Autonomous → Teleop → Endgame → Post-match notes), with big touch targets and a visible phase timer. Counters, not keyboards.
- **Never lose data.** Every interaction writes to local storage immediately, so a browser crash, a dead battery, or an accidental back-swipe cannot destroy a match's work. Draft entries are recoverable.
- **Explicit submit** with a confirmation summary, then auto-advance to the next assigned match.
- **Undo** on counters, and a visible running summary so the scouter can sanity-check before submitting.

### 6.3 Open questions

- **Q6.1** — Do you want the **assignment system** (schedule + who watches which robot per match), or will scouters manually pick match number and team every time? The assignment system is more work but is the single biggest quality lever.
- **Q6.2** — If yes: how are assignments made? (a) auto round-robin over available scouters; (b) fixed station numbers — scouter at station 1 always watches Red 1; (c) manual drag-and-drop by a lead; (d) all three. Station-based is what most successful teams do.
- **Q6.3** — Should the form be **one long scrolling page** or **paged by phase**? Paged is better for focus, worse for going back to fix something.
- **Q6.4** — Do you want an in-app **match timer** that follows the real match phases and can auto-advance pages? Needs a manual "match started" tap since we can't hook the field.
- **Q6.5** **[RAISED BY ME]** — What happens for a robot that is **a no-show, disabled, or dies mid-match**? I recommend a mandatory top-level status (played / no-show / disabled / broke down at time T) because otherwise a dead robot records zeros and destroys its averages.
- **Q6.6** **[RAISED BY ME]** — Should the app support **landscape mode** for tablets, and does the layout need to be usable **one-handed** on a phone?
- **Q6.7** **[RAISED BY ME]** — Accessibility in an arena: high-contrast/outdoor-readable mode, large text, haptic feedback on taps, screen-dimming prevention. Any of these needed?
- **Q6.8** **[RAISED BY ME]** — Do you want a **practice/training mode** where new scouters can enter fake data against a real form without polluting the database? Very useful in the pre-season.

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

### 7.3 Proposed decisions

- **Local store:** IndexedDB (via Dexie or similar), holding: form definitions and versions, the team list, the match schedule, the user's assignments, all of the current event's entries, plus an **outbox** of unsynced changes.
- **Client-generated UUIDs** for all records, so an offline device can create records that will never collide on upload.
- **Outbox / operation log** pattern: every local mutation is appended as an operation with a client timestamp and a monotonic counter; sync replays operations to the server; the server acknowledges and the client prunes.
- **Conflict policy:** entries are naturally partitioned by (match, team, scouter), so genuine conflicts are rare. Proposal: **last-write-wins at field granularity** for scouting entries, with any conflict recorded and surfaced in a "review conflicts" screen for a lead. For form definitions, admin edits while offline are **not allowed** — form changes require connectivity (this avoids the nightmare case of two divergent form schemas).
- **Bounded local data:** don't try to cache all seasons on a phone. Cache the active event plus a configurable window.
- **Soft deletes** everywhere (`deleted_at`), because a delete that happened offline needs to propagate and win predictably.

### 7.4 Open questions

- **Q7.1** — Is the **QR-code transfer** fallback in scope for v1, or a later phase? (I lean: design for it, build it in phase 2 unless your venues really have no internet at all — in which case it's phase 1.)
- **Q7.2** — Do you have any realistic connectivity at your events (Israeli district events specifically)? Phone data? Venue WiFi? This directly decides Q7.1.
- **Q7.3** — Which entities must be **editable offline**? My proposal: scouting entries yes; pick list yes; notes yes; form definitions no; user management no; dashboard creation — probably read-only offline, view cached dashboards but don't build new ones. Agree?
- **Q7.4** — Should **statistics and charts be computed on-device while offline** (so the strategy lead can rank teams with no internet — arguably the single most valuable moment of the whole weekend), or is offline read-only-raw-data acceptable? Computing offline means the metric engine must run in the browser as well as in SQL. This is a significant architectural fork — I want an explicit answer.
- **Q7.5** — When two devices edited the same entry, who wins: latest timestamp, the lead's version, or ask a human? And should we keep the loser for inspection?
- **Q7.6** **[RAISED BY ME]** — Do you want **device-to-device sync on a local network** (no internet, one laptop as hub), or only device→cloud?
- **Q7.7** **[RAISED BY ME]** — How much local storage can we assume? Photos are the risk — 6 robots × 40 teams of pit photos will not fit comfortably. Should photos be deferred-upload-only and never cached in bulk?

---

## 8. Realtime (live updates, no refresh)

### 8.1 Confirmed requirements

- **Optimistic local UI** — a user's own action appears instantly on their own device, no manual refresh (except while offline). *(confirmed 2026-08-14)*
- **Cross-device live push is deferred to nice-to-have** (Q15.1, §24). Online devices see others' changes on the normal refresh triggers — re-querying on screen entry, pull-to-refresh, a manual refresh, and re-sync on reconnect — with optional lightweight polling on specific live screens. Needing a full app restart to see changes is a caching bug to design out (PWA: cache the shell, network-first for data).

### 8.2 Proposed decisions

- **Supabase Realtime** (Postgres logical replication over WebSocket) for live updates on: scouting entries for the active event, the pick list, assignments, and form definition changes.
- **Optimistic UI**: the local write is applied instantly, the server confirms asynchronously, and a failure rolls back with a visible message. This is what makes it feel instant even on bad networks — and it's the same mechanism the offline mode uses, so we build it once.
- A clear connection-state indicator with three states: live / syncing / offline.
- Subscriptions are scoped to the active event to keep bandwidth sane.

### 8.3 Open questions

- **Q8.1** — What genuinely needs to be live? Candidates: the entry feed during a match, aggregate rankings, the pick list during alliance selection, scouter assignment changes, admin form edits. Making *everything* live is expensive; making the *right* things live is cheap. My suggestion: pick list and assignments are live-critical; rankings can refresh every 15–30 seconds; historical browsing needs no live at all.
- **Q8.2** — Should there be a **live "match in progress" view** for the strategy lead showing the six scouters' entries filling in as the match happens? Fun and useful, but requires connectivity that you may not have.
- **Q8.3** **[RAISED BY ME]** — Do you want **presence** (who is online, which scouters have submitted for the current match, who is missing)? This is how you catch "station 4 hasn't submitted for 3 matches" *during* the event rather than after it.
- **Q8.4** **[RAISED BY ME]** — Do you want **notifications** (e.g. "your next match is in 4 minutes", "you have unsynced data", "conflict needs review")? In-app only, or push?

---

## 9. Statistics engine & computed metrics

### 9.1 Confirmed requirements

- Statistics per game/form, viewable per competition or across all competitions of a year.
- Statistics feed the graph/visualisation pages.

### 9.2 Proposed decisions

**Two layers of statistics.**

*Layer 1 — Metric definitions (created in the app, per form):* a named, reusable calculation over form fields.

- Formula over fields: `total_pieces = auto_pieces + teleop_pieces`.
- Aggregation over a team's entries: average, median, sum, min, max, count, standard deviation, percentile, rate (`count where condition / total`), trend/slope over match number, "last N matches" average, best/worst.
- Filters: exclude no-shows and disabled robots; only qualification matches; only a chosen date range or event set.
- Metrics are versioned alongside forms and are reusable across every chart, table and ranking.

*Layer 2 — Derived FRC statistics from official results (if we import them):* OPR, DPR, CCWM, win rate, average ranking points, schedule strength. These come from match scores, not from your scouting, and are a valuable cross-check. Worth having if we do topic 14.

**Aggregation scopes:** per team per event, per team per season, per team all-time, per scouter (for reliability), per match.

**Where computation happens:** SQL views/functions in Postgres for the online case (fast, close to the data), and a shared TypeScript implementation for the offline case — *if* Q7.4 says offline stats are needed. If both exist, they must be driven from the same metric definition and tested against each other, otherwise they will silently disagree.

### 9.3 Open questions

- **Q9.1** — What statistics do you *actually* use to make decisions? Please walk me through how you currently pick alliance partners or plan a match strategy. This tells me which metrics must be first-class rather than "buildable if you configure it".
- **Q9.2** — Do you want a **formula language** for computed metrics (e.g. typing `auto + teleop * 2`), or only a pick-from-a-menu builder (choose field, choose aggregation, choose filter)? A formula language is far more powerful and considerably more work — and it needs a safe evaluator, not `eval`.
- **Q9.3** — Do you want **OPR/DPR/CCWM/EPA**-style computed statistics from official results? (Requires topic 14.)
- **Q9.4** **[RAISED BY ME]** — When several scouters record the same (team, match) — deliberately, for redundancy, or by accident — how do we produce one number? Options: average them, take the most trusted scouter, take the latest, or flag the disagreement for review. This must be decided or your averages will double-count.
- **Q9.5** **[RAISED BY ME]** — **Consistency matters as much as average** in FRC. A team averaging 20 with a standard deviation of 3 is a better partner than one averaging 22 with a deviation of 15. Do you want variance/consistency to be a first-class part of ranking, not just an available metric?
- **Q9.6** — Do you want **recency weighting** (later matches count more, since robots improve and drivers learn)? Simple options: last-N average, or exponentially weighted average.
- **Q9.7** **[RAISED BY ME]** — Do you want **percentile / z-score normalisation** so metrics from different events (with different competition strength) can be compared fairly across a season?
- **Q9.8** **[RAISED BY ME]** — Minimum sample size: should the UI refuse to rank or visibly flag a team with fewer than N observations? Two matches of data presented as a confident average is how bad picks happen.

---

## 10. Dynamic dashboards, graphs & visualisations

### 10.1 Confirmed requirements

- Create statistics and graph pages **dynamically from inside the app**.
- Choose from multiple kinds of graphs, visualisations and tables.
- Point at specific fields in the form, and connect those fields to the charts.
- View the data scoped to one competition, or to all competitions in the same year.

### 10.2 Proposed decisions

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

**Chart types (draft):** bar (grouped/stacked), horizontal bar, line (over match number — the progression chart), scatter (two metrics against each other, for finding complementary partners), radar/spider (multi-metric team comparison — very popular in FRC), box plot (distribution and consistency), histogram, heatmap (field position data, or team × metric), pie/donut (use sparingly), data table with sorting and inline sparklines, and single-number "stat cards".

**Dashboards** are ordered collections of charts on a responsive grid, saved with a name, scoped to a season, and shareable by link. A dashboard has global filters (event, match type, team subset) that all its charts inherit unless overridden.

**Builder UX:** pick a chart type → pick the dimension → add one or more series (field or metric + aggregation) → add filters → live preview → save. Aim for "understandable by a 15-year-old in 5 minutes", not Tableau.

**Standard pages that should exist out of the box** (rather than requiring everyone to build them from scratch):

- Team page: all metrics for one team, match-by-match table, progression charts, notes, photos.
- Ranking page: sortable table of all teams by any metric.
- Compare page: 2–6 teams side by side (radar + table).
- Match preview: the six robots of an upcoming match with predicted contributions.
- Coverage/quality page: which matches are missing data.

### 10.3 Open questions

- **Q10.1** — Which chart types from the list do you actually want in v1? (I'd start with bar, line, scatter, radar, box plot, and table — that covers ~95% of real FRC analysis.)
- **Q10.2** — Are dashboards **personal** (each user builds their own) or **shared team assets** (a lead publishes them for everyone)? Or both, with a "publish" action?
- **Q10.3** — Should a dashboard be **pinned to a season/form**, and what happens to it next year when the form changes completely? Options: it breaks visibly with a "field no longer exists" message; it's archived read-only; or a migration wizard lets you re-map fields to the new form. (I'd do: archived read-only + optional re-map.)
- **Q10.4** — Do you need **export**: chart as PNG, table as CSV/Excel, dashboard as PDF? Printed pick lists and printed match previews are still very common in pits.
- **Q10.5** **[RAISED BY ME]** — Charts on a **phone**: a 40-team bar chart is unreadable on a 6-inch screen. Should the mobile view automatically simplify (top-N, horizontal orientation, table fallback), or should some dashboards be desktop-only?
- **Q10.6** **[RAISED BY ME]** — Which charting library? Recharts (React-native, simple, good defaults), ECharts (very powerful, heavier, best for heatmaps and big datasets), or Visx/D3 (maximum control, most work). I lean Recharts for v1 with an escape hatch, unless you want heatmaps and large scatter plots, which push toward ECharts.
- **Q10.7** **[RAISED BY ME]** — Do you want **drill-down** — click a bar for team 1577 and land on that team's page, click a point on a line chart and see that specific match's raw entry? It's the difference between a pretty chart and an actual analysis tool.
- **Q10.8** **[RAISED BY ME]** — Should charts be able to compare **your scouted data against official results** (e.g. scouted contribution vs OPR) to validate your scouting?

---

## 11. Search, ranking & browse

### 11.1 Confirmed requirements

- Search for teams, forms, ranking, and statistics. (Details to be defined here.)

### 11.2 Proposed decisions

- **Global search** (one search box, keyboard-shortcut accessible on desktop): type a team number or name → team page; type an event name → event page; type a form name → form; type a scouter name → their submissions. Fuzzy matching, recent items, works offline against the cached dataset.
- **Team browse**: filterable, sortable list of all teams at an event or in a season, with a compact metric summary per row and multi-select for comparison.
- **Ranking**: sortable by any metric, with the scope selector (event / season / all-time), sample-size warnings, and a "weighted composite score" option where you assign weights to several metrics — this is effectively how a pick list gets seeded.
- **Entry browse**: raw scouting entries with filters (event, match, team, scouter, form version, flagged), for auditing and fixing.
- Every list view should be shareable by URL with its filters encoded, so a lead can send "look at this" to another device.

### 11.3 Open questions

- **Q11.1** — Do you want a **composite weighted ranking** (assign weights to several metrics to produce one score)? It's the most-used feature of good scouting apps.
- **Q11.2** — Should search reach into **free-text notes** (full-text search over comments)? Useful — "who was mentioned as having a fast climb?" — and needs a Postgres text index, plus a decision about how it works offline.
- **Q11.3** — Do you want to see a team's **multi-season history** (how team 1577 performed in 2024, 2025, 2026) even though the games differ? Only certain things compare across years (reliability, driver skill, general strength) — do you want that view?
- **Q11.4** **[RAISED BY ME]** — Should the team page pull in **public info from TBA** (awards, event history, robot photos, links)? Cheap to add if we're already integrating.

---

## 12. Alliance selection & pick list

### 12.1 Why I'm raising this **[RAISED BY ME]**

You didn't mention it, but it's the reason scouting exists. Everything the app collects over two days is spent in a 20-minute window where a lead must respond in seconds to "team 254 just got picked, who's next?". If the app doesn't support that moment, the data doesn't get used.

### 12.2 Proposed decisions

- An ordered **pick list** per event, built by dragging teams, seeded from any ranking or composite score.
- **Do-not-pick list** with a required reason.
- Separate **first-pick** and **second-pick** lists (different criteria: a first pick complements you, a second pick is often defensive or specialised).
- **Live cross-off**: as alliances form, picked/captain teams are struck through automatically so the next available team is always obvious. Realtime, multi-device, so several people can watch the same list.
- Notes per team visible inline, and printable/exportable for the pit.

### 12.3 Open questions

- **Q12.1** — Is the pick list in scope, and for which phase?
- **Q12.2** — Should alliance selection results be **entered manually** during selection, or imported from the API (which may lag by minutes — too slow for live use)? Probably: manual entry, live-synced across devices.
- **Q12.3** — Do you want **collaborative editing** of the pick list (multiple leads reordering simultaneously) or single-editor with viewers?

---

## 13. Data quality, integrity & scouter reliability

### 13.1 Why I'm raising this **[RAISED BY ME]**

Scouting apps don't fail because of bad charts. They fail because 30% of the matches are missing, one scouter recorded the wrong robot, and someone entered 47 game pieces when the maximum possible is 9 — and nobody noticed until the pick list was being made. Detection has to be part of the product.

### 13.2 Proposed decisions

- **Coverage matrix**: a matches × robot-stations grid for the event showing which cells have data, which are missing, and which are duplicated. This is the lead's most-used screen after the ranking.
- **Validation at entry**: per-field min/max and cross-field rules defined in the form builder (e.g. "scored pieces cannot exceed pieces acquired"), warning vs. hard block configurable.
- **Outlier flagging**: an entry whose values fall far outside the team's or the event's distribution gets flagged for review rather than silently averaged in.
- **Cross-check against official results** where available: your scouted alliance total vs the real score.
- **Scouter reliability score**: per scouter, how often their entries are outliers, disagree with a redundant scout, are missing, or are submitted suspiciously fast. Used gently — for training, and optionally for weighting.
- **Full history on entries** (who edited, when, old value) rather than destructive overwrites.

### 13.3 Open questions

- **Q13.1** — Do you want the coverage matrix and outlier flagging in v1? (I'd argue the coverage matrix is v1 — it's cheap and it saves weekends.)
- **Q13.2** — Do you want **redundant scouting** (two scouters on the same robot) supported as a deliberate feature, with agreement measurement?
- **Q13.3** — Is a **scouter reliability score** something you want, or is it socially awkward for your team? (Legitimate concern — some teams find it demotivating.)
- **Q13.4** — Should validation rules be able to **block** submission, or only warn? (Blocking loses data when reality is weirder than the rule.)
- **Q13.5** **[RAISED BY ME]** — Do you need **bulk fix tools** for an admin (reassign an entry to the correct team, merge duplicates, delete a bad scouter's shift, shift all match numbers by one after a schedule change)? Schedule changes and mis-assigned robots happen at every event.

---

## 14. External data integration (TBA / FRC Events API)

### 14.1 Why I'm raising this **[RAISED BY ME]**

Manually typing 40 team numbers and 100 matches into an app at 8am on competition day is both miserable and error-prone, and it's unnecessary: **The Blue Alliance API** and the official **FRC Events API** provide the event list, team list, match schedule, and match results for free. This one integration removes most manual setup and enables schedule-driven assignment (topic 6) and result validation (topic 13).

### 14.2 Proposed decisions

- Server-side integration (API keys never in the browser) with a scheduled/triggered sync that imports: seasons, events, teams per event, match schedules, match results, and optionally team awards and photos.
- Imported data is stored locally in our own tables, so the app works offline and doesn't depend on a third party being reachable at the venue. Sync runs beforehand, at the hotel.
- A manual fallback: paste/upload a schedule, or enter matches by hand, in case the API is unavailable or you're at an unofficial scrimmage.

### 14.3 Open questions

- **Q14.1** — Do you want this integration? (Strong recommendation: yes.) TBA, the official FRC Events API, or both?
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

**What is the server actually for?** This needs your explicit sign-off, because there's a real design fork. Supabase can be called directly from the browser with row-level security, which is what makes realtime and offline sync clean. So I propose a **hybrid**:

| Path | Used for |
|---|---|
| Client → Supabase directly (RLS-protected) | All normal reads, realtime subscriptions, scouting entry writes. Fast, live, and works with the offline/sync model. |
| Client → Server → Supabase (service role) | Anything privileged or heavy: creating/altering forms and generating their views, user provisioning and role changes, TBA/FRC API imports, bulk fixes, exports, expensive aggregations, scheduled jobs. |

The alternative — *everything* through the server — gives you one control point and easier auditing, but you lose Supabase Realtime's simplicity and add latency on the hot path. I recommend the hybrid.

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

- **Q15.1** — Do you accept the **hybrid** access model, or do you want all traffic to go through your server API?
- **Q15.2** — Client framework: **Vite + React SPA** (simplest, best PWA/offline story, fastest dev loop) or **Next.js** (SSR, routing conventions, one framework for both apps, but its offline/PWA story is more awkward). I lean Vite + React for the client precisely because offline is a core requirement.
- **Q15.3** — Server framework: Hono (tiny, fast, great on Vercel Functions), Express (familiar), or Next.js API routes? And do you want **tRPC** for end-to-end typed calls between client and server?
- **Q15.4** — TypeScript everywhere, I assume? Confirm.
- **Q15.5** **[RAISED BY ME]** — Do you want **generated database types** from Supabase (so a schema change surfaces as a compile error), and **Zod** validation shared between client and server?
- **Q15.6** **[RAISED BY ME]** — Where do **file uploads** (robot photos) go — Supabase Storage, presumably? And who is allowed to upload?
- **Q15.7** **[RAISED BY ME]** — Do you need a **local development** setup that works without touching production data (Supabase local via Docker, or a separate dev project)? I strongly recommend a separate dev Supabase project at minimum — you do not want to test a "delete form" feature against real competition data.
- **Q15.8** **[RAISED BY ME]** — Do you accept the **use-case / transport-agnostic service layer**? It costs a little discipline up front and is the difference between MCP being a day of work and a month of work. This is my strongest recommendation in this whole section.
- **Q15.9** **[RAISED BY ME]** — Should the **MCP server be its own deployable app** (`apps/mcp`) or a route inside the existing server app (`apps/server/mcp`)? A separate app gives independent scaling, its own auth surface and cleaner logs; a route is less to maintain. I lean toward a route in the server app now, extractable later, since they share the same core.

---

## 16. UI/UX design system & responsive behaviour

### 16.1 Confirmed requirements

- Must look good and work well on both phone and computer screens.
- **App chrome is English and LTR** (kept simple); **form content is Hebrew** — labels and free-text notes (Q16.2). Full-app RTL is **not** required, but every text node that can hold Hebrew (form labels, notes, and their appearance on chart axes, tables and team cards) must be bidi-correct (`dir="auto"`). Built in from the start, not retrofitted. *(confirmed 2026-08-14)*

### 16.2 Proposed decisions

- **Two distinct experiences, one codebase**: *phone = data entry* (huge touch targets, minimal chrome, one task per screen, thumb-reachable actions); *desktop = analysis* (dense tables, multi-panel dashboards, keyboard shortcuts, hover detail). These are not the same UI scaled — they're different information densities, and I'd design them as such.
- Tailwind CSS + a component library (shadcn/ui) for speed and consistency.
- **Dark mode**, defaulting to dark: scouting happens in dim arenas, and it saves phone battery.
- Layout: mobile-first, with breakpoints at roughly phone / tablet-landscape / desktop.
- Loading states everywhere, and skeletons rather than spinners for lists.
- An unmistakable, always-visible **offline/sync indicator** — this is a primary UI element, not a footnote.

### 16.3 Open questions

- **Q16.1** — Do you have **team branding** (colours, logo, team number) to build in?
- **Q16.2** — **Language**: English only, Hebrew only, or both? If Hebrew, we need **right-to-left layout support** from the start — retrofitting RTL is genuinely painful, and it affects every component, chart axis, and table. Please answer this early. **[RAISED BY ME]**
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
| Concurrent users | (Depends on Q1.1.) |
| Security | RLS on every table; no service-role key in the browser; audit log on privileged actions. |
| Backup | Automated export of all data per season; the ability to restore. |
| Testing | Unit tests for the metric engine and sync logic; integration tests for the sync protocol; the offline path must be tested, since it's the one that's hardest to debug at a venue. |

### 17.2 Open questions

- **Q17.1** — Are those performance targets right, and is the data-scale estimate roughly correct for your team?
- **Q17.2** **[RAISED BY ME]** — What's your **backup and export** requirement? At minimum I'd want a one-click "export this event/season to CSV+JSON" so a weekend's work can never be lost to a bug, an accidental delete, or a Supabase free-tier surprise.
- **Q17.3** **[RAISED BY ME]** — Which **Supabase plan** are you on? The free tier pauses inactive projects and caps database size and realtime connections — a paused project on competition morning would be very bad. Worth checking before we design around it.
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
Auth and roles → events and teams → form builder with core field types → offline data entry → sync → raw data browse → basic per-team stats and ranking table.

**Phase 2 — Analysis**
Metric builder → chart and dashboard builder → team compare → coverage/quality matrix → export.

**Phase 3 — Competition workflow**
TBA/FRC API import → scouter assignments → live views and presence → pick list and alliance selection.

**Phase 4 — Depth**
Pit and super scouting forms → photos → advanced field types (position picker, event log, cycle times) → scouter reliability → multi-season history → QR fallback transfer.

**Phase 5 — MCP server (DEFERRED, not currently in scope)**
Expose the existing use cases as MCP tools, resources and prompts → auth for the MCP endpoint → connect an MCP client and iterate on tool descriptions → evals.

**Phase 6 — In-app AI (DEFERRED, not currently in scope)**
Server-side LLM orchestration → in-app chat/insights panel → cached briefing documents → notes summarisation → natural-language-to-chart.

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

---

## 21. Decision log

Decisions get recorded here as topics close, so Claude Code (and future you) can see not just what we chose but why.

| Date | Topic | Decision | Rationale |
|---|---|---|---|
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

---

## 22. Open questions index

Quick reference for what's still unanswered. Answered questions get struck through and moved into the relevant section as a confirmed requirement.

**Closed 2026-08-14:** Q1.4, Q3.1, Q3.2, Q3.9, Q7.4, Q15.1, Q15.2, Q15.4, Q15.8, Q16.2 → confirmed requirements. Q14.1 → §24 nice-to-have.

- **Topic 1:** Q1.1 – Q1.6
- **Topic 2:** Q2.1 – Q2.7
- **Topic 3:** Q3.1 – Q3.10
- **Topic 4:** Q4.1 – Q4.5
- **Topic 5:** Q5.1 – Q5.8
- **Topic 6:** Q6.1 – Q6.8
- **Topic 7:** Q7.1 – Q7.7
- **Topic 8:** Q8.1 – Q8.4
- **Topic 9:** Q9.1 – Q9.8
- **Topic 10:** Q10.1 – Q10.8
- **Topic 11:** Q11.1 – Q11.4
- **Topic 12:** Q12.1 – Q12.3
- **Topic 13:** Q13.1 – Q13.5
- **Topic 14:** Q14.1 – Q14.5
- **Topic 15:** Q15.1 – Q15.9
- **Topic 16:** Q16.1 – Q16.6
- **Topic 17:** Q17.1 – Q17.5
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

All seven top blockers are now closed. Next-tier durable questions still open: Q3.7 (delete-form semantics), Q9.4 (duplicate-scout resolution), Q6.5 (dead-robot status), Q5.2 (scouter login).

---

## 23. Change history

Every edit is logged here so changes can be audited without reprinting the document. See `COLLABORATION.md` section 6 for how to review diffs properly.

| Version | Date | Sections touched | Change |
|---|---|---|---|
| v0.1 | 2026-08-12 | all | Base document created from the initial requirements. 19 topics, ~100 questions. Added the fixed/flexible domain split (topic 2), venue-connectivity reality (topic 7.2), form versioning (topic 3.3), external API import (topic 14), pick list (topic 12) and data quality (topic 13) as items not in the original brief. |
| v0.2 | 2026-08-12 | 0.2, 3.2, 3.3, 3.4, 15.2, 15.3, 19.1, 22 | Added topic 20 (AI/LLM/MCP). Added semantic field metadata to 3.3 with Q3.9–Q3.10. Added transport-agnostic use-case layer to 15.2 with Q15.8–Q15.9. Noted that the MCP goal strengthens the JSONB recommendation in 3.2. Added phases 5–6. |
| v0.3 | 2026-08-12 | 0.2, 19.1, 20.1, 20.4, 21, 22 | **Scope decision:** AI/MCP is setup only — no LLM connection. Phases 5–6 marked deferred. Q20.1–Q20.4 and Q20.7–Q20.12 parked; Q20.5–Q20.6 remain live. First Decision Log entry added. |
| v0.4 | 2026-08-12 | 23 | Added this change history section. Companion file `COLLABORATION.md` created (process rules, answer shorthand, session templates, review workflow, build-phase rules). |
| v0.5 | 2026-08-12 | — | No spec changes. `COLLABORATION.md` v1.1 adds section 10 (working in Claude Code: surface choice, `CLAUDE.md`, `/clear` discipline, plan mode, review loop, permissions). `CLAUDE.md` created for the repo root. |
| v0.6 | 2026-08-14 | 0, 1.1, 3.1, 7.1, 8.1, 15.1, 16.1, 21, 22, 24 (new) | Closed the seven top blockers plus Q3.2/Q15.1/Q15.2/Q15.4: storage = Option A (JSONB); immutable versioning (label/range edits in place); semantic metadata required (desc/unit/phase/direction); offline stats yes; single-team; English LTR chrome + Hebrew bidi form content; all-traffic-through-server; TypeScript everywhere (React+Vite / Node+Hono / shared engine); runtime-generated form validation. Cross-device realtime and TBA/FRC import moved to new §24 nice-to-have. Decision Log rows added. `nice` shorthand added to `COLLABORATION.md` §3 and `CLAUDE.md`. |

---

## 24. Nice-to-have (wanted, deferred)

Confirmed as **wanted but deliberately out of current scope** — revisit later. Distinct from *parked* (topic 20 — timing undecided pending a separate decision) and *skip* (out of scope permanently). Shorthand `Q_ nice` moves an item here.

| Item | Source | Note |
|---|---|---|
| **External data import (TBA / FRC Events API)** — event, team, schedule and result import | Q14.1, topic 14 | Wanted, not now. Deferring it also defers schedule-driven **scouter assignments** (Q6.1/Q6.2) and **official-result validation** (topic 13), which depend on it. |
| **Cross-device live updates (Supabase Realtime)** — another device's changes appearing without a refresh | Q15.1, topic 8 | Optimistic local UI and refresh-on-triggers ship now; automatic cross-device push is deferred. Additive later given the all-through-server model. |
