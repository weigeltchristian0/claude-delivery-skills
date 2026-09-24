# PR body & Jira write-back templates

## Pull request — opened ready for review
Open with `mcp__bitbucket__createPullRequest` — **ready for review, not a draft**. One PR **per environment branch**: source = that environment's worktree branch (`…`, `…-staging`, `…-main`), destination = the matching environment branch (`develop` / `staging` / `main`). Add reviewers from `mcp__bitbucket__getEffectiveDefaultReviewers`.

### Title
`{{PROJECT_KEY}}-####: <concise what>`
(No `[DRAFT]` suffix — the PR is opened ready for review.)

### Description template — keep it SHORT
A reviewer should grasp a bot PR in ~20 seconds. **One line per item, plain language, no prose padding, don't restate the diff. Omit any line that doesn't apply** (no risks → no Risks line). Aim for ≤ ~8 lines total. The status header carries the verification result, so there is no separate verification section.

First line is the **status header** — exactly one of `✅ Verified` / `⚠️ Unverified`, plus the one-line check result:

```
✅ Verified — build ✓ · lint ✓ · tests ✓ (142)

**Ticket:** [{{PROJECT_KEY}}-####](https://{{JIRA_HOST}}/browse/{{PROJECT_KEY}}-####) — <title>
**Change:** <1–2 lines: what changed and why>
**ACs:** <AC1 → where; AC2 → where — flag any AC not covered>
**Risks:** <migration/dep/infra/deletion flag · out-of-scope · disclosed self-review finding — omit line if none>
**Siblings:** <multi-repo only — links to the other PRs>
```

Status header rules:
- `⚠️ Unverified` states the reason inline: `⚠️ Unverified — phpunit ✗ (2 failing)` or `⚠️ Unverified — toolchain not runnable; CI will check`.
- High-risk changes (migration / dependency bump / infra / large deletion) are **always `⚠️`** even on green: `⚠️ Unverified — DB migration, needs human eyes`.
- The **`main` (production) PR is always `⚠️`** — a customer-facing production deploy needs human eyes even on green and even when already proven on develop/staging: `⚠️ Unverified — production deploy of <change>; confirm before merge`.

**Siblings — cross-link every related PR.** For each repo, the develop/staging/main PRs are siblings of each other; for a multi-repo ticket, the other repos' PRs are siblings too. A promotion PR's body should state it carries the same change as the develop PR (applied cleanly or hand-ported because the branch had drifted). Open all PRs, then make sure each one's **Siblings:** line links the others (update after the last is created).

## Jira write-back (default ON; skip on `--no-jira`)
Post **one new comment** per run with `mcp__atlassian-remote__addCommentToJiraIssue` using `contentFormat: "markdown"` — never edit a prior comment. Team convention: plain words, **no code blocks or backticks**, PR link(s) + verdict, **the branch name(s) created**, **date after the verdict**.

**Always state every branch name you created** — one per environment per repo — as plain text (e.g. `Branch feature/{{PROJECT_KEY}}-2017-abschluss-min-gap-main`). A branch name is a plain identifier, not "code/backticks" — write it WITHOUT backticks so it respects the plain-words convention and stays copy-pasteable. This is the durable record people read straight from the ticket, instead of digging through the Bitbucket development panel. Also note any environment that already carried the change and so got no PR.

**PR links MUST be Markdown links, never a bare URL.** Jira's markdown→ADF conversion does not auto-link a raw `https://…` — it renders as plaintext (not clickable). Always write `[PR #<id>](<url>)` (or `[{{CORE_SERVICE}} PR](<url>)` for multi-repo). A Markdown link is not "code/backticks", so it still respects the plain-words convention.

Example (single repo, verified):
```
Implemented and opened a PR for review on branch feature/{{PROJECT_KEY}}-2017-abschluss-min-gap. Verified — build, lint and tests pass. [PR #1291](https://bitbucket.org/{{WORKSPACE}}/{{CORE_SERVICE}}/pull-requests/1291) — reviewer needs to read, approve and merge. 2026-06-22.
```
Example (all three environments, one already present):
```
Opened PRs into all three environments for {{CORE_SERVICE}}. Branches: feature/{{PROJECT_KEY}}-2017-abschluss-min-gap on develop, feature/{{PROJECT_KEY}}-2017-abschluss-min-gap-staging on staging, feature/{{PROJECT_KEY}}-2017-abschluss-min-gap-main on main. Develop and staging already carried the change, so PRs were opened only where it was missing. Build passes; the main PR is flagged for review as a production deploy. [develop PR](https://bitbucket.org/{{WORKSPACE}}/{{CORE_SERVICE}}/pull-requests/1291), [staging PR](https://bitbucket.org/{{WORKSPACE}}/{{CORE_SERVICE}}/pull-requests/1292), [main PR](https://bitbucket.org/{{WORKSPACE}}/{{CORE_SERVICE}}/pull-requests/1299) — please review and merge in order develop, staging, main. 2026-06-30.
```
Example (multi-repo, one unverified):
```
Implemented across {{CORE_SERVICE}} and {{FRONTEND_REPO}}. Branches: {{CORE_SERVICE}} on feat/{{PROJECT_KEY}}-1234-slot-reason, frontend on feat/{{PROJECT_KEY}}-1234-slot-reason. Two PRs opened and cross-linked: [{{CORE_SERVICE}} PR](https://bitbucket.org/{{WORKSPACE}}/{{CORE_SERVICE}}/pull-requests/1291), [frontend PR](https://bitbucket.org/{{WORKSPACE}}/{{FRONTEND_REPO}}/pull-requests/842). Frontend verified; {{CORE_SERVICE}} unverified because two tests fail, flagged in the PR. 2026-06-22.
```
Example (blocked, no PR):
```
Did not implement — the ticket is too underspecified to code safely. Open questions: <1–2 lines>. No branch or PR created. 2026-06-22.
```

Keep it to 2–4 sentences. The branch name carries {{PROJECT_KEY}}-#### so the PR also auto-links under the ticket's development panel — but state the name in the comment too, so it's readable without opening that panel.
