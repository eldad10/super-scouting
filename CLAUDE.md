# FRC Scouting Platform

Year-agnostic scouting app for an FRC team. Currently in the **specification phase** — no application code exists yet.

## Read these before doing anything

- `docs/spec/frc-scouting-app-spec.md` — the living specification. Single source of truth for all requirements and decisions.
- `docs/spec/COLLABORATION.md` — process rules. **Sections 2, 3 and 10 are binding.** Read them.

## Non-negotiables

1. **No application code until `SPEC-FINAL.md` and `IMPLEMENTATION-PLAN.md` exist and are approved.** Right now the only job is closing spec topics.
2. **Use plan mode before every non-trivial edit.** Plan, get approval, then execute one task.
3. **One task at a time.** Stop after each so I can review the diff and commit.
4. **Verify before claiming done.** Run the command, read the output, then report. Never say something works without having seen it work.
5. **Never touch the production Supabase project.** Development work uses the dev project only.
6. **Migrations live in the repo** and are applied by CLI. Never hand-edit a schema in the Supabase dashboard.
7. **Secrets stay in server environment variables.** Never in the client, never committed.

## Communication

- Be terse. No preamble, no recaps, no restating documents back to me.
- One recommendation, not a trade-off table — unless I write `?` or `why?`, or the decision is irreversible.
- Never reprint spec sections. Patch the file and give a change summary of five lines or fewer.
- **Raise omissions proactively.** If I'm forgetting something that will hurt later, say so unasked. Mark it `[RAISED BY ME]` in the spec. This is the most valuable thing you do here.
- Flag contradictions with closed decisions before editing, not after.

## How I answer spec questions

Shorthand, defined in `COLLABORATION.md` section 3. Summary: `Q3.1 A` = pick option A · `y`/`n` = yes/no · `?` = explain first · `you pick` = use your recommendation and log it · `later` = park it · `skip` = out of scope · `nice` = add to the nice-to-have list (spec §24).

## When a spec topic closes

Set its status to CLOSED in the status table, move proposals into confirmed requirements, add a Decision Log row **with rationale**, add a Change History row.

## Project gotchas

Full list in `COLLABORATION.md` section 9. The ones that bite hardest:

- Venue connectivity is effectively zero — offline is the normal operating mode, not an edge case.
- Form versions are immutable; field keys are permanent, labels are not.
- Semantic field metadata cannot be backfilled — capture it at field-creation time.
- A dead or no-show robot must not record zeros.
