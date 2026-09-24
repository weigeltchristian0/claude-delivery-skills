---
name: Bug Hunter
user-invocable: true
description: Triage and root-cause a bug report from a symptom (not a Jira ticket), grounded in your repos, code architecture, and the read-only the data warehouse prod DWH. Use whenever someone pastes or describes a bug — e.g. a Slack message like "customers can't see their meetings", "double bookings are showing up again", "the slot suggestions are empty for shop X", "{{IDENTITY_SERVICE}} 500s on role update" — and wants to know WHAT is broken, WHERE in the code, WHY, and WHO should fix it. Maps the symptom to the likely service/repo, reads the current code on the latest branch, optionally confirms the data pattern and blast radius in the data warehouse, and returns a concise Slack-pasteable triage with ranked root-cause hypotheses. Distinct from `code-explorer`/`qa-code-review` (those review a known PR/ticket); this one works backwards from a symptom. Optionally files an {{PROJECT_KEY}} bug with `--ticket`.
arguments:
  - name: report
    description: The bug report — free text pasted from Slack, an email, a screenshot description, etc. If omitted, ask for it.
    required: false
  - name: deep
    description: "If set (--deep), fan out specialist sub-agents (per-repo code tracers, a data-grounding agent, a challenger) for hard/ambiguous bugs. Default OFF (fast single-pass)."
    required: false
  - name: ticket
    description: "If set (--ticket), after triage hand the findings to the `ticket-writer` skill to file an {{PROJECT_KEY}} Bug. Default OFF."
    required: false
---

# Bug Hunter — Symptom → Root Cause Triage

## Purpose
A bug report tells you a **symptom** ("X is broken"), not a cause. This skill works backwards: it localizes the symptom to the right service/repo, reads the **current** code, optionally confirms the data pattern and blast radius in the prod DWH, and hands back a concise, Slack-pasteable triage with **ranked root-cause hypotheses** — each with concrete evidence (file:line or a data figure) and a confidence level — plus the recommended next step and likely owning team.

It is deliberately **honest about uncertainty**: a bug report is rarely enough to be sure. Rank hypotheses, show your evidence, and say what you'd need to confirm — a confident-but-wrong root cause wastes more time than an honest "most likely X, here's how to confirm."

## Operating principles
- **Concise above all.** This output gets pasted into a Slack thread. One line per hypothesis, no filler, no restating the report back at length. (Matches the team's terse-reporting norm.)
- **Simple, plain language.** Short sentences, common words, no marketing phrasing — every line must add information.
- **Evidence beats assertion.** Every hypothesis cites a file:line, an event/queue, or a DWH figure. No evidence → mark it a guess.
- **Localize before you read.** Don't grep 61 repos. Use the architecture map to pick the 1–3 likely services first, then read those.
- **Hunt on the latest code.** Always refresh the suspect repos to their latest default branch before reading — you're hunting a live bug, not reviewing a change.
- **Ground in data when state is involved**, but never confuse prod (DWH) with stage.

---

## Workflow

### Phase 0 — Parse the report (you do this directly)
Extract from the free-text report and restate as a **one-line symptom statement**:
- **What** is wrong (the observable behaviour vs. expected).
- **Where/who** — any concrete entity: user/customer/meeting ID, email, shop, role, URL, screen.
- **When** it started / how often / is it new or a regression.
- **Environment** — prod or stage? (Affects whether the DWH can ground it — see Phase 3.)
- **Severity hints** — how many users, data loss, money, blocking?

List anything critical that's **missing** — you'll surface these as "open questions for the reporter" rather than guessing.

### Phase 1 — Localize to the architecture (you do this directly)
Using the **Architecture & feature→repo map** below (and `references/repo-catalog.md` for the full repo index), pick the **1–3 most likely repos/services** and the suspect code path: which API endpoint, event-bus message, queue consumer, frontend route, or scheduled job would produce this symptom. State your routing reasoning in one line.

If the symptom is cross-service (e.g. "meeting shows in app but not in {{INTEGRATION_SERVICE}}"), name the **boundary** (the event/sync between them) as a prime suspect — integration seams are where these bugs live.

### Phase 2 — Refresh & open the suspect repos (you do this directly)
For each suspect repo, **always get to the latest state first** (you're hunting current behaviour):

**Base directory:** `C:\Users\ChrisWeigelt\Documents\Claude Code\`
Local path = `<BASE>\<repo-slug>`; remote = `git@bitbucket.org:{{WORKSPACE}}/<slug>.git`.

1. **If the local clone exists:** refresh it to the latest default branch:
   `git -C "<path>" fetch origin --prune` → determine default branch (`git -C "<path>" symbolic-ref --short refs/remotes/origin/HEAD` → e.g. `origin/main`) → `git -C "<path>" checkout <branch>` → `git -C "<path>" pull --ff-only origin <branch>`.
   - If the working tree is dirty (a leftover PR branch from `code-explorer`), `git -C "<path>" stash` first so the pull is clean; note it but don't lose their work.
2. **If it doesn't exist:** `git clone git@bitbucket.org:{{WORKSPACE}}/<slug>.git "<path>"`.
3. **If clone/refresh fails** (huge repo, no access, offline): fall back to the **Bitbucket MCP** — read files via `mcp__bitbucket__getRepository` + the file/source tools, or search PRs/commits for the suspect area. Note in the output that you used remote read (less complete than local grep).

Do repo prep in parallel (one Bash call per repo). Read code with **Read/Grep/Glob** (no approval needed) — not Bash `cat`/`grep`.

### Phase 3 — Hunt for the cause

**Default (lean, single-pass):** you trace it yourself. Read the suspect path, follow the logic from entry point → data layer, and form ranked hypotheses. Look for the usual culprits: unhandled null/empty, enum/casing mismatch, wrong filter (e.g. soft-deleted rows), state-transition race, missing auth/permission branch, broken cross-service event/contract, timezone math, off-by-one, recently-changed code (`git -C "<path>" log --oneline -15 -- <suspect file>` to see if a recent commit lines up with "when it started").

**Optimizer / slot-rejection symptoms** ("user not suggested", "no slots for day X", "does not respect driving time") — the solver validates the **entire day**, so triage the day, not the reporter's segment:
1. **Audit the full existing day chain first.** Pull the user's complete calendar for the date (`APP_MEETING` incl. type `Pause`, plus coordinates), then feasibility-check **every consecutive leg** with the engine's real arithmetic: drive time (OSRM as approximation) + padding (`drive × 10% + 8 min` per leg — `{{ENGINE_SERVICE}} Config.kt` / `optimizer.padding_*` settings) + break duration when a Pause sits in the gap. A pre-existing infeasible leg poisons every candidate slot for the whole day.
2. **Reason codes are many-to-one.** One UI message can come from several hard constraints (e.g. "Meeting does not respect driving time" = `scoreOverlaps` OR `scoreOverlapsBreak` — see `ScoreCalculator.violatedHardConstraints`). Enumerate ALL constraints mapping to the message and test each against data before ranking hypotheses.
3. **Check which constraints fire from existing state alone.** Constraints without a `newRequest` filter (e.g. `scoreOverlapsBreak`) make the day infeasible regardless of the new meeting — check these first when the symptom is "nothing the user changes helps".
4. **Distrust minute-thin hypotheses.** If the leading explanation only works when estimates line up within a minute or two, keep hunting — a robust cause (a leg short by half an hour or more) usually exists.

**`--deep` (fan out sub-agents)** — for hard or ambiguous bugs, launch in parallel:
- **One code-tracer per suspect repo** — "trace the path for symptom S from entry point to data layer; report the single most likely defect with file:line and why."
- **A data-grounding agent** (Phase 3.5) — runs the the data warehouse checks.
- After they report, **one challenger** — "attack these hypotheses: which are false positives, what did they miss, re-rank by real-world likelihood." Give each agent the symptom statement + suspect repo paths + the instruction to use Read/Grep/Glob locally.

### Phase 3.5 — Ground in data (the data warehouse, optional but encouraged)
If the symptom involves **state, counts, or 'how many affected'**, confirm it against reality instead of guessing. A read-only prod DWH mirror may exist at `{{DWH_DIR}}/`:
- **Read `{{DWH_DIR}}/SCHEMA_GUIDE.md` first** (schema map, `<table prefix>` tables, enum casing, soft-deletes via `<soft-delete column>`, UTC-naive timestamps). Query from that dir: `python -c "from db import run; print(run('select ...'))"` (SELECT-only; the guard blocks writes).
- Use it to: **confirm the data pattern actually occurs** (is the bad row/enum/null real?), **quantify blast radius** (how many rows/users/meetings, and **since when** via `max(<sync timestamp column>)`/created dates), and sanity-check the hypothesis.
- **Caveat — DWH is PROD, the app QA target is stage (a different DB).** If the report is about **prod**, the DWH grounds it directly. If it's about **stage**, the DWH cannot confirm stage state — use it only to learn realistic data shapes, and say so. Never paste prod PII into the output.
- **⚠️ The DWH is NOT real-time — it syncs on a delay (often hours).** A "row not found" / zero count — especially for an entity created or edited **today** (exactly the case for a fresh bug report) — usually means *not synced yet*, NOT that it doesn't exist, and NOT that it's stage-only. Before concluding anything from a miss, check `select max(<sync timestamp column>) from prod.AIRBYTE.<TABLE>;`: if the entity's creation time is after that, it's just unsynced. Never downgrade a prod report to "stage" on a DWH miss alone.
- If absent or a query errors, **skip silently** — grounding is a bonus, never required.

### Phase 4 — Concise triage output (you do this directly)
Synthesize into the Slack-pasteable template below. Resolve `--deep` agent conflicts with your judgment; dedupe; rank by likelihood × evidence strength.

### Phase 5 — File a ticket (only if `--ticket`)
When `--ticket` is set, after presenting the triage, **invoke the `ticket-writer` skill** (via the Skill tool) to file an {{PROJECT_KEY}} **Bug**. Hand it everything you already found so it does NOT re-investigate:
- Call it as `--type=Bug` and pass a brief containing: the one-line symptom, the concrete examples (the reporter's scenarios), the **ranked technical findings with file:line refs** (your root cause + other hypotheses), the suggested fix, the affected repos, and any reference IDs/URLs.
- **Explicitly tell ticket-writer NOT to re-run code analysis** — bug-hunter already did the deep dive; it should use the findings as-is. This avoids redundant work and the sync-lag/"doesn't exist" trap.
- Keep the technical notes in **plain language** (explain the cause in words; keep file/repo names in parentheses for developers). ticket-writer drafts → confirms with the user → creates, and places business-outcome Acceptance Criteria in `{{AC_FIELD_ID}}`.
- Relay ticket-writer's final result (the `{{PROJECT_KEY}}-####` key + URL) back in your output.

---

## Output format

ALWAYS use this structure. Keep it tight enough to paste into Slack. Omit empty sections.

```
🐛 *<one-line symptom statement>*
Scope: <severity + blast radius — e.g. "HIGH · ~120 prod customers since 2026-06-08" or "unknown, needs repro">
Env: <prod | stage | unknown>

*Most likely cause* (confidence: High/Med/Low)
<repo · file:line> — <one-line explanation of the defect> <(+ data evidence if any)>

*Other hypotheses*
- <repo · file:line> — <one-liner> (conf: M/L)
- <repo · file:line> — <one-liner> (conf: M/L)

*Data check* (only if the data warehouse was used)
<one line: what you confirmed + the figure, e.g. "4 overlapping Approved meetings for user X in prod, all since the 06-08 deploy">

*Next step*
<the single most useful action — e.g. "reproduce on stage with meeting <id>", "check the the event bus consumer in {{CORE_SERVICE}}", "owner: {{CORE_SERVICE}} team">

*Open questions for reporter* (only if blocking)
- <what you need to confirm/repro>
```

**Example (abridged):**
Input: "Hey, since yesterday a bunch of customers in one region are seeing meetings that overlap — like two appointments at the same time on the same {{SPECIALIST_ROLE}}. Prod."
Output:
```
🐛 *Overlapping Approved meetings appearing for the same {{SPECIALIST_ROLE}} (one region, prod, since ~yesterday)*
Scope: HIGH · double-booking, customer-facing · need count
Env: prod

*Most likely cause* (confidence: Med)
{{CORE_SERVICE}} · src/meeting/availability.service.ts:~140 — overlap check filters by STATUS='Approved' but not by the soft-delete/cancel flag, so a re-book against a slot freed by a cancelled meeting isn't blocked.

*Other hypotheses*
- {{ENGINE_SERVICE}} · solver constraint — stale availability snapshot offered an already-taken slot (conf: M)
- {{CORE_SERVICE}} · the event bus meeting.created consumer — race between two near-simultaneous bookings (conf: L)

*Data check*
Confirmed in prod DWH: 6 overlapping Approved, non-deleted meeting pairs for one region users, all START_MEETING ≥ 2026-06-10 (matches "since yesterday").

*Next step*
Pull `git log` on availability.service.ts around the 06-10 deploy; owner: {{CORE_SERVICE}} team.
```

---

## Architecture and feature-to-repo map

**Read `ARCHITECTURE.md` at the root of this skills directory.** It names the
services, their stacks, the events and queues between them, and which feature
lives where. Fill it in once per organisation; every skill reads the same file.

Routing rule of thumb: pick the one to three most likely services before
reading any code, and when a symptom crosses a boundary, suspect the boundary
itself. Integration seams are where these bugs live.

## Jira (for `--ticket`)
- Project key: `{{PROJECT_KEY}}`. Cloud ID: `{{JIRA_HOST}}`. Filing is delegated to the `ticket-writer` skill (drafts → confirms → creates).

## Guidelines
- One line per hypothesis. Rank them. Show evidence. Mark confidence honestly.
- Don't fabricate a file:line — if you couldn't pin it, say "area: <module>" and make pinning it the next step.
- Recent commits are gold for regressions — when the report says "since yesterday", check `git log` dates against the deploy.
- If you genuinely can't localize it, say so and list what repro/data you'd need — that's a useful answer, not a failure.
