---
name: QA Orchestration
user-invocable: true
description: Run unattended/headless end-to-end QA of completed development tickets ({{PROJECT_KEY}}) on a VM or via `claude -p`. Use this whenever the user wants to QA one or more tickets automatically, run a scheduled/cron QA pass, or process a list of ticket IDs end-to-end. A THIN orchestrator — it gathers each ticket's context once, then sequences the three QA sub-skills (qa-code-review → qa-test-cases → qa-execute) per `--phases`, and emits a one-line-per-ticket summary. Contains no review/generation/Jira logic of its own. Renamed from the former `qa-tickets-auto`.
arguments:
  - name: ticket_ids
    description: One or more Jira ticket IDs, space- or comma-separated (e.g., "{{PROJECT_KEY}}-1861" or "{{PROJECT_KEY}}-1861, {{PROJECT_KEY}}-1799").
    required: true
  - name: phases
    description: "Which phases to run, comma-separated: review,cases,execute (default: review,cases — execute only when explicitly requested, since it drives a browser)."
    required: false
  - name: jira
    description: "If set (--jira), forward --to-jira to each sub-skill it runs (review/cases write back to Jira). Default OFF. The orchestrator itself never touches Jira."
    required: false
---

# QA Orchestration — Headless Sequencer

## Purpose
Sequence the three focused QA sub-skills across a list of tickets, unattended. This skill is **thin**: it gathers shared context once per ticket and delegates all real work. It owns no review logic, no test-case logic, and **no Jira logic** — Jira write-back lives entirely in the sub-skills (the orchestrator only forwards a flag).

- **qa-code-review** — multi-agent review → dated code-review sub-page.
- **qa-test-cases** — generate/classify → ticket page + library; Execution = Auto/Manual.
- **qa-execute** — run the `Execution = Auto` cases on stage → Execution Run sub-page.

## Language
- Write in simple, plain language. Short sentences, common words.
- No filler, no preamble, no marketing phrasing. Every line must add information.
- One idea per line; prefer bullets over paragraphs.

## Headless operation contract
1. **Never ask the user anything.** Missing info or a failed step → record it and continue.
2. **Each ticket is independent.** A failed ticket gets a partial result; never abort the whole run.
3. **Tool calls run without approval** (intended env: `bypassPermissions` on a VM).
4. **Always produce output on Confluence** via the sub-skills, even on partial/failed tickets.
5. **End with a one-line-per-ticket summary** to stdout.

## Flags
- `--phases=review,cases,execute` — default **`review,cases`**. Include `execute` only when the caller wants the browser run too (it's non-destructive but drives Playwright on stage).
- `--jira` — forward `--to-jira` to each sub-skill that runs. Default **off**. The orchestrator performs no Jira calls itself; it only passes the flag through.

## Phase 0: Environment & base directory
- **Base dir `<BASE>`:** `QA_BASE_DIR` env var, else current working dir.
- **Date `<DATE>`:** system date `YYYY-MM-DD` at runtime.
- **Repo map** (under `<BASE>`): `{{FRONTEND_REPO}}`, `{{CORE_SERVICE}}`, `{{IDENTITY_SERVICE}}`, `app`, `{{ENGINE_SERVICE}}`, `{{CUSTOMER_SERVICE}}`.
- **Optional DWH:** if `{{DWH_DIR}}/db.py` exists, a read-only **prod** data-warehouse mirror is available; the sub-skills (qa-code-review / qa-test-cases) may use it to ground findings/cases in real data (see their "Data warehouse" sections + `{{DWH_DIR}}/SCHEMA_GUIDE.md`). The orchestrator does nothing with it directly — it's a pass-through capability. Note its presence/absence in the bundle so sub-skills know whether to attempt it; never block a ticket on it.
- **Parse `ticket_ids`** into a clean deduped uppercase list.

## Per-ticket repository preparation (MANDATORY — run before Phase 2)
Do this **before sequencing the sub-skills**, after the PR links are known (Phase 1 step 2). If you skip it, the working tree is whatever was left checked out from earlier work (often an unrelated ticket branch or a stale/detached HEAD), and every source cross-reference silently degrades to "diff-only" — the failure mode that makes a review unable to confirm runtime behaviour.

For each involved repo (from PR repos / labels):
- If `<BASE>/<slug>` exists: `git -C "<path>" fetch origin`, then **check out the PR's `source_branch`** (from `jira_get_pr_links`, e.g. `{{PROJECT_KEY}}-1658-onbording`, `feature/{{PROJECT_KEY}}-1986-staging`): `git -C "<path>" checkout <source_branch> && git -C "<path>" pull --ff-only`. If that branch was deleted after merge, check out the **destination** branch (`develop`/`staging`/`main`) and `pull` so the merged commit is present.
- Else clone `git@bitbucket.org:{{WORKSPACE}}/<slug>.git` and check out as above.
- **Verify the checkout actually contains the ticket's changes:** `grep`/`Glob` for a distinctive new symbol or file from the diff (e.g. a new function name or added file path). If it's absent, the branch is wrong or unfetched — fix it, or note "source browsing unavailable; diff-only review" and continue.

Record, per repo, the **checked-out branch + commit** and whether it was verified, and put that in the context bundle so the sub-skills (and their agents) know the local source is real and current, not stale. If git access fails entirely, note diff-only and continue.

## Phase 1: Gather the context bundle ONCE (you do this directly)
Per ticket, gather once and pass to the sub-skills so they don't re-fetch:
1. **Ticket:** `mcp__atlassian__jira_get_issue` (`fields="*all"`, `comment_limit=50`).
2. **PR links:** `mcp__jira-dev-status__jira_get_pr_links` (fallbacks: labels → repo; branch `fix/{TICKET}-*` / `feat/{TICKET}-*`; PR URLs in comments).
3. **PR diffs:** `mcp__bitbucket__getPullRequestDiff` per PR (parallel; large diffs saved to a file — read fully).
4. **PR details:** `mcp__bitbucket__getPullRequest` per PR.
5. **Comments & screenshots (first-class input — do not skip).** Treat the ticket's Jira comments as a source of truth alongside the AC: they carry reported bugs, repro steps, clarified/changed expected behaviour, and QA feedback. Distill them into a short **comment digest** for the bundle. Then **download every image attachment on the ticket and look at each one** — any `attachment` whose `mimeType` starts with `image/`, including images pasted into comments (all land in the issue `attachment` field under `fields="*all"`): authenticated GET to the attachment `content` URL with the same Atlassian email + API token the MCPs use (from the user-scope `mcpServers` env in `~/.claude.json`; Basic auth = base64(`email:token`)), save to `<BASE>/qa-tickets-reports/<DATE>/attachments/<TICKET>-<n>-<filename>`, then `Read` the file so you actually see it. (curl: `curl.exe -s -L -H "Authorization: Basic <b64>" -o "<path>" "<content-url>"`.) For each, write a one-line description of what it shows (bug repro / mockup / error state / expected UI) into the bundle next to its local path. Usually a handful; if there are many (>8), analyse those referenced by the description/AC/comments first and note any left unanalysed. If credentials or a download fail, note "attachments not analysed (<reason>)" and continue — never block a ticket on it.
Compile the bundle: ticket summary/description/AC/**comment digest**/labels/assignee; **the analysed screenshots (one-line description + local path each)**; each PR's title/author/branches/full diff; local repo paths **and the branch + commit checked out per repo (from repo prep) plus whether it was source-verified**; the organisation's architecture block (see qa-code-review). If no PRs are found, still run qa-test-cases against the AC + comments + screenshots (it can note "no PRs") and record it.

## Phase 2: Sequence the sub-skills (per ticket, in order)
Run only the phases in `--phases`. Order matters — test-cases consumes the review's findings:
1. **review** → invoke **qa-code-review** with the context bundle (and `--to-jira` if `--jira`). Capture the verdict + findings.
2. **cases** → invoke **qa-test-cases** with the context bundle + the review findings (and `--to-jira` if `--jira`). Capture #cases, #Auto, #promoted, ticket-page URL.
3. **execute** → invoke **qa-execute** for the ticket (and `--to-jira` if `--jira`). Capture the run tally + run-page URL. (Only if `execute` is in `--phases`.)
A sub-skill failure for one ticket → record the partial result and move on.

## Phase 3: Final run summary (stdout)
One line per ticket, e.g.:
```
QA run <DATE>:
  {{PROJECT_KEY}}-1861 → DEPLOY WITH FOLLOW-UPS → 14 cases (3 promoted, 5 Auto) → <ticket page url>
                run `qa-execute {{PROJECT_KEY}}-1861` to execute the 5 Auto cases
  {{PROJECT_KEY}}-1799 → HOLD → 6 cases (2 promoted, 1 Auto) → <url>
  {{PROJECT_KEY}}-1801 → FAILED (no PRs found) → ticket page written with explanation
```

## Guidelines
- Thin orchestrator: gather → sequence → summarize. No review/generation/Jira logic here.
- Concise summary, one line per ticket.
- Project key `{{PROJECT_KEY}}`; cloud `{{JIRA_HOST}}`; workspace `{{WORKSPACE}}`.
- For interactive / hands-on work, invoke a sub-skill directly (e.g. `qa-code-review {{PROJECT_KEY}}-1861`) instead of this orchestrator.
