---
name: QA Code Review
user-invocable: true
description: Multi-agent code review of a completed development ticket ({{PROJECT_KEY}}). Use when the user wants ONLY a code review of a ticket/PRs (verdict + findings) — no test-case generation, no browser run. Runs 4 specialist agents + 2 challengers over the ticket's PR diffs, synthesises a deploy verdict (DEPLOY AS-IS / WITH FOLLOW-UPS / HOLD), and writes a dated code-review sub-page under the ticket's Confluence QA page. Optionally posts a summary comment to the Jira issue with `--to-jira`. One of the three QA sub-skills sequenced by `qa-orchestration`; runs standalone too.
arguments:
  - name: ticket_id
    description: The Jira ticket ID to review (e.g., {{PROJECT_KEY}}-1861). Space/comma-separated list also accepted when run standalone.
    required: true
  - name: to-jira
    description: "If set (--to-jira), also post a NEW dated summary comment on the Jira issue (one per run). Default OFF."
    required: false
---

# QA Code Review — Multi-Agent Review → Dated Sub-Page

## Purpose
Produce ONLY a code-review verdict for a completed ticket and write it to a **dated code-review sub-page** under the ticket's Confluence page. This is the review half of the QA system, split out of the old `qa-tickets-auto`. Test-case generation lives in **qa-test-cases**; browser execution in **qa-execute**; sequencing in **qa-orchestration**.

This skill writes **no local output files** — its deliverable is the Confluence sub-page. It may download the ticket's image attachments to a scratch dir in order to analyse them (see Context bundle). It never opens a browser.

## Flags
- `--to-jira` (default **off**): in addition to the Confluence sub-page (always written), post a NEW dated summary comment on the Jira issue (one per run). See "Jira write-back".

## Confluence coordinates (`QA` space)
- Cloud `{{JIRA_HOST}}` · Space `QA` (id `{{QA_SPACE_ID}}`).
- **Ticket QA** parent `{{TICKET_PARENT_ID}}` — per-ticket pages are children titled `{{PROJECT_KEY}}-xxxx — <summary>`. The code-review sub-page is a child of that ticket page (create the ticket page first if it doesn't exist yet — a stub with the title is fine; qa-test-cases fills its body).
In practice the token-based `mcp__atlassian__confluence_*` storage tool is usually **not** loaded — use `mcp__atlassian-remote__*` (the canonical Atlassian MCP; prefer it over the deprecated `claude_ai_Atlassian` connector), which accepts `html`/`markdown`/`adf` only (no `storage`). Prose (verdict panel, bullets) can go via `html`/`markdown`. **Tables are different — full-width is mandatory and `html` cannot express it.** A `<table>` written via `html` always renders at `default` (narrow/centered) layout — the html format has no full-width attribute, and an `html` readback won't reveal the problem. **So write every table you include (requirements matrix, edge-case table, etc.) with `contentFormat: "adf"`:** set each `table` node's `attrs.layout = "full-width"` and include **no** fixed column widths (`colwidth`/`<colgroup>`/`width`) — fixed widths make this MCP drop the layout. **Verify via an `adf` read** (every `table` must be `full-width`, no `colwidth`), never an `html` read. (Storage tooling, if present: `content_format: "storage"`, `<table data-layout="full-width">`, no `<colgroup>`.)

## Context bundle
When sequenced by `qa-orchestration`, the orchestrator passes a prepared **context bundle** — use it verbatim and skip re-fetching. When run standalone, gather it yourself:
1. **Ticket:** `mcp__atlassian__jira_get_issue` (`fields="*all"`, `comment_limit=50`, cloud `{{JIRA_HOST}}`).
2. **PR links:** `mcp__jira-dev-status__jira_get_pr_links`. Fallback: repo from labels (frontend→{{FRONTEND_REPO}}, backend→{{CORE_SERVICE}}/{{IDENTITY_SERVICE}}, {{INTEGRATION_SERVICE}}→app, customer→{{CUSTOMER_SERVICE}}); branch `fix/{TICKET}-*` / `feat/{TICKET}-*`; PR URLs in comments.
3. **PR diffs:** `mcp__bitbucket__getPullRequestDiff` per PR (parallel). Large diffs are saved to a file path — read it fully in chunks.
4. **PR details:** `mcp__bitbucket__getPullRequest` per PR.
5. **Check out the ticket's code (do this — a source-aware review beats diff-only).** Don't assume the working tree is on the right branch; it's usually left on an unrelated ticket branch or a stale/detached HEAD. Per involved repo: `git -C "<BASE>/<slug>" fetch origin`, then `git checkout <source_branch> && git pull --ff-only` (`<source_branch>` from the PR links; if it was deleted after merge, check out the merged **destination** — `develop`/`staging` — instead). **Verify** with `grep`/`Glob` that a distinctive new symbol or file from the diff is present; if it isn't, the tree is stale → fix the branch, or fall back to diff-only and say so on the sub-page. When sequenced by `qa-orchestration`, repo prep already ran and the bundle carries the checked-out branch+commit — trust that instead of re-checking out. Record, per repo, the branch/commit and whether source was verified.
6. **Comments & screenshots — first-class input, not just a requirements source.** Mine the ticket's Jira comments (reported bugs, repro steps, clarified/changed expected behaviour, QA feedback) into a short **comment digest**. Then **download and look at every image attachment** on the ticket — any `attachment` with an `image/*` mimeType, including images pasted into comments (all present in the `attachment` field under `fields="*all"`): authenticated GET to each `content` URL with the same Atlassian email + API token the MCPs use (user-scope `mcpServers` env in `~/.claude.json`; Basic auth = base64(`email:token`)) → save to `<BASE>/qa-tickets-reports/<DATE>/attachments/<TICKET>-<n>-<filename>` → `Read` it so you actually see it. (curl: `curl.exe -s -L -H "Authorization: Basic <b64>" -o "<path>" "<url>"`.) Record a one-line description per image (bug repro / mockup / error state / expected UI) with its local path. If credentials/download fail, note "attachments not analysed (<reason>)" and continue. When sequenced by `qa-orchestration`, the bundle already carries the comment digest + analysed screenshots — use them, don't re-fetch.
Compile: ticket summary/description/AC/**comment digest**/labels/assignee; **the analysed screenshots (one-line description + local path each)**; each PR's title/author/branches/full diff; local repo paths (`<BASE>/<slug>`) **and the checked-out branch+commit + verified-or-not**; the architecture block below. If no PRs are found, write a sub-page noting "no PRs to review" and stop.

Include in every agent prompt:
> Local repos at `<BASE>/<repo-slug>`, checked out to this ticket's PR branch (`<branch>@<commit>`, source-verified). Use `Read`/`Grep`/`Glob` for source context. **First confirm a known changed symbol from the diff is actually present in the working tree** — if it isn't, the checkout is stale, so review diff-only and state that. The live PR diff always wins over local source. Do NOT use Bash for file ops.
> The bundle also carries the ticket's **comment digest** and any **screenshots we analysed** (one-line description + local file path each). Treat comments as authoritative alongside the AC — a comment may report a bug, change the expected behaviour, or carry QA feedback the diff must satisfy. Use the screenshots to understand the reported problem and the expected UI; you may `Read` a screenshot's file path for pixel detail.

The deploy verdict must state which mode it was: **source-verified** (local tree confirmed on the ticket branch) or **diff-only** (carry the confidence caveat — unconfirmed runtime-behaviour findings stay VERIFY, not confirmed).

## Phase A: Parallel analysis (4 agents in one message)

#### Agent 1: Code Quality Reviewer
```
You are a senior code reviewer doing QA on a completed ticket. ONLY code quality (not edge cases, tests, requirements).
CONTEXT: {context_bundle}
TASK: Analyze every diff line. Per finding: Severity (CRITICAL/CONCERN/SUGGESTION), file:line, issue, why it matters, fix. Focus: logic/bugs, security (auth/exposure/injection), performance (N+1, hot paths), patterns/consistency, architecture (cross-service, API contracts), error handling.
OUTPUT — concise:
### Code Quality Findings
- **[SEVERITY] Title** (file:line) — issue + fix.
### Code Quality Summary
1-2 sentences, counts per severity.
```

#### Agent 2: Edge Case Analyst
```
You are a QA edge case specialist. ONLY edge cases/boundaries/failure scenarios.
CONTEXT: {context_bundle}
TASK: For each change consider null/undefined, empty collections, boundaries, type mismatches, races, network failures, state transitions, permission boundaries, data integrity, browser/env (PWA/offline). Per case: scenario, status (HANDLED/UNHANDLED/PARTIAL), risk (HIGH/MED/LOW), mitigation.
OUTPUT — only UNHANDLED/PARTIAL:
### Edge Cases
| # | Scenario | Status | Risk | Mitigation |
### Edge Case Summary
1-2 sentences, highest-risk only.
```

#### Agent 3: Test Coverage Assessor
```
You are a QA test coverage specialist. ONLY test coverage.
CONTEXT: {context_bundle}
TASK: Which tests added/modified? Quality of existing (scenarios, assertions, isolation, mocking)? What SHOULD be tested (happy/error/authz/integration/transform)? Prioritize missing: CRITICAL/IMPORTANT/NICE.
OUTPUT:
### Existing Tests
One line per file. "None" if absent.
### Missing Test Coverage
- **[PRIORITY] Scenario** — why. Only CRITICAL/IMPORTANT.
### Test Coverage Summary
1-2 sentences.
```

#### Agent 4: Requirements Verifier
```
You are a QA requirements specialist. ONLY requirements vs implementation.
CONTEXT: {context_bundle}
TASK: Extract every requirement (AC, description, comments, screenshots/mockups, title) — a comment that changes the expected behaviour and a mockup that pins the expected UI are requirements too. Per req: PASS/PARTIAL/FAIL/CANNOT VERIFY + evidence. Flag scope creep + cross-service consistency.
OUTPUT:
### Verification Matrix
| # | Requirement | Status | Evidence |
Add **Scope Analysis** only if real scope creep.
### Requirements Summary
1-2 sentences, PARTIAL/FAIL only.
```

## Phase B: Challenge round (2 agents in one message)
Compile the 4 reports into one block, then:

#### Challenger A: Devil's Advocate
```
Challenge the 4 specialists' findings. ORIGINAL CONTEXT: {context_bundle}  SPECIALIST FINDINGS: {combined}
TASK: false positives? overstated severity? wrong assumptions? duplicates? Missed anything (re-read diffs)? Validate each CRITICAL/CONCERN's real-world impact.
OUTPUT:
### Challenges
- **[ref]** KEEP/DOWNGRADE/UPGRADE/REMOVE + reason
### Missed Findings  (new only)
### Severity Adjustments  (changes only)
```

#### Challenger B: Pragmatic Reviewer
```
Make the report actionable/proportionate. ORIGINAL CONTEXT: {context_bundle}  SPECIALIST FINDINGS: {combined}
TASK: Filter for signal (prod incident/data loss/security/blocks users vs theoretical). Prioritize ALL into MUST FIX / SHOULD FIX / CONSIDER / DROP. Flag false alarms / standard codebase patterns. Deploy readiness.
OUTPUT:
### Prioritized Findings
**MUST FIX / SHOULD FIX / CONSIDER / DROPPED**
### Deploy Recommendation
DEPLOY AS-IS / DEPLOY WITH FOLLOW-UPS / HOLD — one-line justification.
```

## Phase C: Synthesize → dated Confluence sub-page
Synthesize the 6 reports: resolve conflicts (honor good challenger REMOVE/UPGRADE), dedupe, set final severity (pragmatic-led, devil's-advocate-adjusted) and the deploy verdict. **Create a code-review sub-page** (`confluence_create_page`, parent = the ticket page id, title `<TICKET_ID> · Code Review — <DATE>`, `<DATE>` = system date `YYYY-MM-DD`). Each run creates a new dated sub-page so history is preserved.

**Write for a non-engineer — simple, short, skimmable.** This is the most important rule for the page (the agents above work in code terms; the page must not). Concretely:
- **Everyday words, not code.** Say "the cancel screen doesn't record who cancelled," not "completeMeeting omits cancelled_by from the repo.update payload." No jargon, method names, or class names in prose; at most one inline `code` token (e.g. an endpoint path) when it genuinely helps. Spell out impact in business terms.
- **Plain status words, not severity codes.** In the verdict and findings use words like **Blocker / Must fix / Should fix / Note** — not CRITICAL/CONCERN. In the requirements table use **Done / Partly / Done, untested** — not PASS/PARTIAL/FAIL.
- **Short.** One sentence per finding where possible; a bold lead-in + one or two plain sentences at most. Aim for ~4–6 findings, not a catalogue — fold low-value items into a single line or drop them. Table notes are one short clause each.
- **Lead with the bold takeaway.** Each bullet starts with a bold plain-language label, then the consequence, then the fix.

Structure:
1. Verdict panel — Deploy as-is / Deploy with follow-ups / Hold + one plain sentence on why (+ a short italic line on review mode: source-verified vs diff-only, and whether a PR existed).
2. What we found — a few bullets, each a bold plain label + the impact + the fix.
3. **Code evidence** — one collapsible `expand` per blocker/must-fix finding (skip pure notes), titled in plain language, containing the exact code behind the finding. Lead the section with an italic one-liner ("For the developers — the exact code behind each finding."). Inside each expand: one short plain sentence per snippet naming `repo/path/file.ts:line` (inline `code` mark), then a language-tagged `codeBlock` (`typescript`, `diff`, …) with the relevant lines — short excerpts (≤15 lines), trimmed to the offending code, with a brief `// ←` annotation on the key line where it helps; use `diff` blocks to show removed/changed guards. Quote real source from the verified checkout (or the PR diff), never paraphrased code. The expands keep the page skimmable: prose stays non-technical, code lives only inside the collapsed sections.
4. Requirements — short table (Requirement / Status / Note), plain status words, one-clause notes.
5. Tests to add — short bullets noting what `qa-test-cases` should cover (this skill does not write cases).

Keep the remaining technical depth (full agent reports, severity codes) in the agent reports and the orchestrator summary — on the page, technical content lives ONLY inside the Code-evidence expands.

Record the verdict for the orchestrator's run summary.

## Jira write-back (only when `--to-jira`)
The Confluence sub-page is always written; this is additive.
- **Resolve to the parent issue.** If `ticket_id` is a **sub-task**, find its parent task/story/bug (`jira_get_issue` → parent field) and act on the parent. Never comment on a sub-task.
- **New comment every run.** Post a NEW bot comment on each run, identified by its leading signature `_QA Code Review_` (italic). NEVER edit or replace a previous run's comment — review history stays in the comment stream, matching the dated Confluence sub-pages. The run date goes in parens right after the verdict, NOT inside the signature. Do NOT use a raw HTML-comment marker (`<!-- ... -->`) — it renders as visible literal text in Jira's ADF.
- **One line only, NO code.** The comment is a SINGLE line in plain words: `_QA Code Review_ — <emoji> **<VERDICT>** (<DATE>): <one short plain-language clause summarising why> (+ MUST/SHOULD items in plain words, only if any). [Full review →](<sub-page url>)`. Emoji ✅ for DEPLOY AS-IS, ⚠️ for WITH FOLLOW-UPS, ⛔ for HOLD. No bullet lists, no multi-paragraph findings — the detail (and all code) lives on the Confluence sub-page.
- **Formatting rules (the markdown→Jira conversion is fragile):** (1) NO backticks/inline code anywhere — no branch names, setting keys, file/method names; each backtick token renders as a code lozenge and shreds the one-liner into fragments. (2) Keep the italic signature EXACTLY `_QA Code Review_` — putting parentheses or anything else inside the underscores breaks the italics and shows literal `_…_`. (3) Plain business language only, as on the Confluence page.
- **Working link.** Post with `contentFormat: "markdown"` and use markdown link syntax `[text](url)` — NOT Jira wiki syntax `[text|url]` (which does not render under markdown). No status transitions, no labels.

## Final output (stdout)
One line: `<TICKET> → <verdict> → <code-review sub-page url>` (+ `· jira-commented` when `--to-jira`).

## Architecture and feature-to-repo map

**Read `ARCHITECTURE.md` at the root of this skills directory.** It names the
services, their stacks, the events and queues between them, and which feature
lives where. Fill it in once per organisation; every skill reads the same file.

Routing rule of thumb: pick the one to three most likely services before
reading any code, and when a symptom crosses a boundary, suspect the boundary
itself. Integration seams are where these bugs live.

## Data warehouse (optional, read-only)
A read-only mirror of the **production** data warehouse may be available at `{{DWH_DIR}}/` (`<BASE>` = `QA_BASE_DIR` or CWD). **Read `{{DWH_DIR}}/SCHEMA_GUIDE.md` first** — it maps the schemas, the `<table prefix>` app-DB tables, join keys, and the gotchas (soft-deletes via `<soft-delete column>`, UTC-naive timestamps, FK lookups, and that **the DWH syncs on a delay — a missing or recent row may just be unsynced, not absent; never conclude an entity doesn't exist from a DWH miss, check `max(<sync timestamp column>)`**). Query it from that dir: `python -c "from db import run; print(run('select ...'))"` (`run()` is SELECT-only). If the folder is absent or a query errors, **skip it silently** — it is never required to produce a verdict.

Use it to **ground a finding's real-world impact** rather than guessing: does the enum value / null / orphaned row a change assumes actually occur in prod, and at what volume? how many rows would this migration or query touch? is the edge case the Edge Case Analyst raised real or theoretical? A finding confirmed against real data can be stated with confidence; one that can't stays `VERIFY`. **Caveat: the DWH is PROD, the QA app is stage — a different DB.** Use it to understand real data shapes, never to assert anything about stage state.

## Guidelines
- The Confluence page is for a non-engineer: simple everyday words, short, skimmable. No method/class names or jargon in prose; plain status words (Done/Partly, Blocker/Must fix/Should fix/Note) not severity codes. One sentence per finding where possible; ~4–6 findings, not a catalogue. Exception: real code snippets belong in the collapsed Code-evidence expands (one per blocker/must-fix) — that's the only place code appears on the page.
- Concise: one line per finding, no filler. Skip "all good" sections.
- This skill does code review only — no test cases, no browser, no Jira unless `--to-jira`.
- Project key `{{PROJECT_KEY}}`; cloud `{{JIRA_HOST}}`.
