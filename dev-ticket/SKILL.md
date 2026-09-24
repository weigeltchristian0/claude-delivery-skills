---
name: Dev Ticket
user-invocable: true
description: Autonomously implement an {{PROJECT_KEY}} Jira ticket end-to-end and open pull request(s) ready for review for a human developer to review, approve, and merge. Use whenever the user wants Claude to actually DO the development for a ticket — e.g. "implement {{PROJECT_KEY}}-2027", "develop {{PROJECT_KEY}}-1234 and open a PR", "pick up this ticket and code it", "build out this ticket". Reads the ticket + acceptance criteria, maps it to the organisation's repo(s) via the architecture map, implements the change once in an isolated git worktree matching each repo's conventions, runs that repo's build/lint/tests, adversarially self-reviews its own diff, then promotes the identical change through the environment branches and opens one PR into each of develop, staging and main (cross-linked, ready for review) carrying a verification status header. The PR is the ONLY human gate. Distinct from `qa-code-review`/`qa-execute` (those review an existing PR/ticket), `ticket-writer` (creates tickets), and `bug-hunter` (triages a symptom into a root cause).
arguments:
  - name: ticket_id
    description: The Jira ticket to implement, e.g. "{{PROJECT_KEY}}-2027". If omitted, ask for it.
    required: true
  - name: repos
    description: "Override repo auto-detection (--repos={{CORE_SERVICE}},{{FRONTEND_REPO}}). Rare — the architecture map usually routes correctly. Use when the ticket is mis-routed or you want to scope the work."
    required: false
  - name: targets
    description: "Override which environment branches to open PRs into (--targets=develop,staging,main). Default: every one of develop/staging/main that the repo actually has. Use to scope down, e.g. --targets=develop to land on the integration branch only."
    required: false
  - name: no-jira
    description: "If set (--no-jira), do NOT post the PR-link comment back to the Jira ticket. Default: Jira write-back is ON."
    required: false
  - name: solo
    description: "If set (--solo), force inline single-context implementation with no sub-agent fan-out, even for multi-repo work. For when you know it's trivial and want speed. Default: auto-scale (see Phase 4)."
    required: false
---

# Dev Ticket — Ticket → PR(s)

## Purpose
Take one {{PROJECT_KEY}} ticket and carry it all the way to **pull request(s) opened ready for review**, which a human reviews, approves, and merges. The skill is fully autonomous: there is **no plan-approval step**. The **PR is the only human gate** — so everything before it (decomposition, isolation, verification, self-review) exists to make that single gate trustworthy.

It implements **across repos** when a ticket spans them (e.g. an API in `{{CORE_SERVICE}}` + UI in `{{FRONTEND_REPO}}`), and within each repo it promotes the change through the organisation's environment branches — opening one cross-linked PR into each of `develop`, `staging`, and `main`. Each PR carries an honest **verification status header** so the reviewer sees state at a glance.

## Language
- Write in simple, plain language. Short sentences, common words.
- No filler, no preamble, no marketing phrasing. Every line must add information.
- One idea per line; prefer bullets over paragraphs.

## Operating principles
- **One PR per environment branch.** the organisation promotes a change through `develop` (integration) → `staging` → `main` (production). The skill implements the change **once** on the development branch, then promotes the **identical** change onto each higher environment branch, opening one ready-for-review PR into each. The bot opens all of them at once; the **human controls promotion timing** by choosing when to merge each (develop first, then staging, then main). (See Phases 3–7.)
- **Every PR opens ready for review.** The bot pushes each branch and opens a normal (non-draft) PR for a human to review, approve, and merge. The bot itself never approves or merges — that is the human gate.
- **Never fake green.** If the toolchain can't run, the change is high-risk, or a check fails, say so in the status header and the "Risks / did NOT do" section. Evidence before assertions.
- **Worktrees only.** Never write code in the main clones — `bug-hunter`/`qa-*` check those out and would collide. Each branch's work happens in an isolated git worktree (Phases 3 & 6).
- **Localize before you code.** Use the architecture map to pick the 1–N target repos; don't touch repos the ticket doesn't need.
- **Match the repo.** Mirror each repo's existing conventions, structure, and test style. Read neighbouring code before adding any.
- **Honest degradation.** If part of a multi-repo ticket is too ambiguous to implement safely, do the clear repos and for the rest either skip the PR or open it with explicit open questions in the body — never guess your way to a merge-ready-looking diff.

---

## Workflow

### Phase 0 — Environment & base directory (you do this directly)
- **Base dir `<BASE>`:** `QA_BASE_DIR` env var, else the current working dir (`C:\Users\ChrisWeigelt\Documents\Claude Code`).
- **Date `<DATE>`:** system date `YYYY-MM-DD` at runtime.
- **Core repo map** (clones under `<BASE>\<slug>`): `{{FRONTEND_REPO}}`, `{{CORE_SERVICE}}`, `{{IDENTITY_SERVICE}}`, `{{CUSTOMER_SERVICE}}`, `{{ENGINE_SERVICE}}`, `{{INTEGRATION_SERVICE}}`. Full 61-repo index: `../bug-hunter/references/repo-catalog.md`.
- **Environment branches:** the organisation repos promote through `develop` → `staging` → `main` (production). The skill targets each of these the repo actually has (treat `master` as the `main` alias). `--targets` overrides the set.
- **Worktree root:** `<BASE>\.dev-worktrees\` (created on demand).
- Normalize `ticket_id` to uppercase `{{PROJECT_KEY}}-####`. If empty, ask for it.

### Phase 1 — Gather ticket context (you do this directly)
1. **Ticket:** `mcp__atlassian-remote__getJiraIssue` — summary, description, **Acceptance Criteria (`{{AC_FIELD_ID}}`)**, labels, issue type, attachments, comments. The AC list is your definition of done.
2. **Existing dev work:** `mcp__jira-dev-status__jira_get_pr_links` and `mcp__jira-dev-status__jira_get_dev_status`. If a branch/PR already exists for this ticket, **do not duplicate** — continue that branch, or stop and report it. This also tells you which environments already carry the change (a merged develop PR means develop is done — Phase 6 will skip it, not re-open it).
3. **Linked spec:** if the ticket links a Confluence design/spec page, fetch it (`mcp__atlassian-remote__getConfluencePage`) — it usually carries the real intent the one-line AC compresses.
4. Note the issue **type** (Bug → `fix/…` branch; Story/Task → `feat/…`).

### Phase 2 — Localize & decompose (the internal plan — NO human gate)
This is the skill's brain. Even though no human approves it, you must produce it explicitly before touching code.
- Use the **Architecture & feature→repo map** below to pick the **target repo(s)** (honour `--repos` if given).
- For **each** target repo, write down: the precise change (files/modules/endpoints/components), the cross-repo **contract** it must honour (API shape, event payload, DTO) so sibling repos line up, and the **test idea** that would prove it.
- Map **every acceptance criterion** to a concrete change in some repo. An AC with no home is a red flag — surface it as an open question rather than inventing scope.
- If the ticket is genuinely too vague to implement any repo safely, **stop here** and post a Jira/terminal note with what's blocking — do not open an empty or speculative PR.

### Phase 3 — Primary worktree on the development branch (you do this directly, per target repo)
Implement the change once, on the lowest environment branch; Phase 6 promotes it upward. Keep the main clones pristine — all writes happen in worktrees.
1. **Refresh the main clone:** `git -C "<BASE>\<slug>" fetch origin --prune`; if the tree is dirty, `git -C ... stash` first (don't lose leftover work).
2. **Resolve the environment branches.** List which of `develop`, `staging`, `main` (or `master`) exist on the remote (`git -C "<BASE>\<slug>" ls-remote --heads origin develop staging main master`, or `mcp__bitbucket__getEffectiveRepositoryBranchingModel`). Intersect with `--targets` if given. The **primary** is the first that exists in order develop → staging → main; the rest are **promotion targets** for Phase 6.
3. **Create the primary worktree + branch** off `origin/<primary>`:
   `git -C "<BASE>\<slug>" worktree add "<BASE>\.dev-worktrees\<slug>\{{PROJECT_KEY}}-####" -b <feat|fix>/{{PROJECT_KEY}}-####-<short-slug> origin/<primary>`
   The develop/primary branch carries **no environment suffix**; promotion branches get a `-staging` / `-main` suffix in Phase 6. The branch name **must contain `{{PROJECT_KEY}}-####`** so `jira-dev-status` auto-links it.
4. From here, **read and write only inside the worktree path.** Record per repo: worktree path, primary branch, primary env, base commit, and the promotion-target list.
5. **Announce the branch name the moment it's created** — emit a one-line `created branch <name> in <repo> → <primary>` so the exact name is visible immediately. The recorded branch names are what later surface in the Phase 9 output **and** the Phase 8 Jira comment — capture every one verbatim.

### Phase 4 — Implement (auto-scaling, on the primary worktree)
Decide depth from the Phase 2 decomposition — do not ask, just scale:
- **1 repo + trivial change** (config, copy, one-liner, single small function) **or `--solo`** → implement **inline** in this context.
- **Multiple repos, or non-trivial** → fan out **one implementer sub-agent per primary worktree, in parallel** (Agent tool). Give each agent: the symptom-free task ("implement these changes in repo X to satisfy ACs a,b"), the worktree path, the contract it must honour, the instruction to match repo conventions and use Read/Grep/Glob/Edit locally, and to add tests where sensible. Agents return a summary of what they changed + any blockers.

In all cases: follow each repo's conventions, keep diffs focused (no drive-by refactors the ticket doesn't need), and add/adjust tests where it clearly makes sense. Leave no debug code, commented-out blocks, or stray TODOs. This is the **canonical** implementation — Phase 6 reuses this exact change, so get it right here.

### Phase 5 — Verify (independently, on the primary worktree)
Run that repo's **real** checks and capture evidence. **The authoritative source for the commands is the repo's CI config** — mirror what CI actually runs. See `references/verification.md` for per-stack detection + commands (Node, PHP, JVM, frontend). Record which commands ran and pass/fail. If the toolchain can't run at all (missing runtime, no deps), mark it **Unverified** — never assume green.

### Phase 5.5 — Adversarial self-review (MANDATORY — never skipped)
Before promoting or opening any PR, review the primary diff as a skeptic would (`requesting-code-review` turned inward). For multi-repo work, run one challenger sub-agent per worktree; for a single small change, do it inline. The challenger attacks the diff:
- Does it actually satisfy **each** mapped acceptance criterion?
- Did it break or drift from a **cross-repo contract**?
- Leftover debug/TODO/commented code? Secrets or credentials introduced?
- Are the **risks correctly flagged** (migration, dep bump, infra, deletion)?
- Does the change match neighbouring code, or stick out?

Each finding is **either fixed** (re-run Phase 5) **or surfaced** in the PR's "Risks / did NOT do" section. Do not proceed to promotion/PRs until every finding is fixed or explicitly disclosed.

### Phase 6 — Promote the change to the other environment branches (you do this directly, per repo)
For each promotion target (the staging/main branches found in Phase 3, in promotion order):
1. **Skip if the env branch already carries the change.** Re-runs and already-merged work are common. Check before doing anything: is the primary commit already an ancestor (`git -C "<BASE>\<slug>" merge-base --is-ancestor <primary-commit> origin/<env>`), or does the target file already show the change? If so, **do not** create a branch or PR for that env — record "already on <env>" and move on. (This is what spares {{PROJECT_KEY}}-style re-runs from empty PRs.)
2. **Create the promotion worktree + branch** off `origin/<env>`: `<feat|fix>/{{PROJECT_KEY}}-####-<slug>-<env>` (e.g. `…-staging`, `…-main`).
3. **Re-apply the identical change.** Cherry-pick the primary commit if it applies cleanly (`git cherry-pick`). Environment branches drift — `main` can be hundreds of lines behind `develop` — so if it conflicts, **hand-port** the change to match that branch's structure, keeping the behaviour **identical** to the reviewed primary diff. Sanity-check parity: the promotion diffstat should match the primary's, and the resulting behaviour must be the same. Never let a promotion silently diverge from what was reviewed.
4. **Re-verify the promotion.** If the touched file was byte-identical to the primary's and the primary built clean, the build carries over — say so honestly. If you had to port, **re-run the build** on this worktree (the surrounding code differs). Record applied-cleanly-vs-ported and the verify result per env.
5. Record per env: worktree path, branch, base commit, ported?, verify result.

### Phase 7 — Open PR(s) (ready for review, one per environment branch)
Per repo, for **each environment branch that has a real change** (primary + every non-skipped promotion target):
1. `git -C "<worktree>" add -A && git commit` (concise message, first line `{{PROJECT_KEY}}-####: <what>`), then `git -C "<worktree>" push -u origin <branch>`.
2. Open the PR **ready for review** with `mcp__bitbucket__createPullRequest` targeting that environment branch — **not** a draft.
3. **Status header** (first line of the description) reflects Phase 5/6:
   - `✅ Verified` — build/lint/tests passed and self-review is clean.
   - `⚠️ Unverified` — a check failed, couldn't run, or the change is high-risk. **High-risk is ALWAYS `⚠️` regardless of green tests**, and **the `main` (production) PR is ALWAYS `⚠️`** — a customer-facing production deploy needs human eyes even when everything is green and it's already proven on develop/staging.
4. Body follows `references/pr-and-jira.md`: ticket link, what/why, **per-AC mapping**, verification evidence, and "Risks / did NOT do / follow-ups". **Cross-link the sibling PRs** — the develop/staging/main PRs for the same repo, plus cross-repo siblings for multi-repo tickets. Promotion PRs state they carry the same change (cleanly applied or ported). Add effective default reviewers (`mcp__bitbucket__getEffectiveDefaultReviewers`).
5. **No empty PRs:** skip any env branch that already had the change (Phase 6.1) or any repo that ended with no real change — note why in the summary.

### Phase 8 — Jira write-back (default ON; skip if `--no-jira`)
Post **one new comment** to the ticket via `mcp__atlassian-remote__addCommentToJiraIssue` — never edit a previous one. Follow the team convention: plain words, **no code/backticks**, the PR links and verification verdict, **date after the verdict**. **Name every branch you created** (one per environment per repo, plain text, e.g. `Branch feature/{{PROJECT_KEY}}-####-<slug>-main` — no backticks), and note any environment that already had the change so the ticket records the full picture. Format in `references/pr-and-jira.md`.

### Phase 9 — Output summary (stdout)
Concise block per repo, one row per environment PR. See Output format.

### Phase 10 — Clean up the worktree(s)
Once the PRs are open the work is safe on the remote, so remove **every** worktree created (up to one per environment per repo) to keep `<BASE>\.dev-worktrees\` from accumulating. **Branches and PRs are unaffected** — `git worktree remove` deletes only the working directory, not the branch.

Order matters, especially on Windows:
1. **Move your shell out of the worktree first.** A shell whose working directory is inside the worktree locks it; removal then fails with "in use" / "Permission denied". `cd` back to `<BASE>` in *every* shell (Bash and PowerShell) you used inside it.
2. **If you junctioned in `node_modules` for verification, delete the junction itself first** with `rmdir "<worktree>\node_modules"` (cmd) — that removes only the link. Never `Remove-Item -Recurse` the worktree while the junction is live, or you can delete the main clone's real `node_modules`. Sanity-check the main clone's dep count is unchanged before/after.
3. `git -C "<BASE>\<slug>" worktree remove --force "<worktree>"` then `git -C "<BASE>\<slug>" worktree prune`.
4. On Windows `git worktree remove` often deregisters the worktree (it leaves `worktree list`) but still fails to delete the folder — if a leftover dir remains, remove it directly afterward (now safe, the junction is already gone).

Skip cleanup (and say so) if a worktree ended **Unverified with no PR** and is worth keeping for the user to inspect.

---

## Output format
Keep it tight — one block per repo, one row per environment PR, no filler.
```
🔧 {{PROJECT_KEY}}-#### — <ticket title>
Repos: <n> · PRs: <m> · Status: <✅ all verified | ⚠️ n unverified>

<repo-slug>
  develop  [✅ Verified | ⚠️ Unverified]  <feat|fix>/{{PROJECT_KEY}}-####-<slug>          → <PR url>
  staging  [✅ | ⚠️]                       <feat|fix>/{{PROJECT_KEY}}-####-<slug>-staging  → <PR url>   (or "already on staging — no PR")
  main     [⚠️ Unverified]                 <feat|fix>/{{PROJECT_KEY}}-####-<slug>-main     → <PR url>   (production → always ⚠️)
  checks:  build ✓ · lint ✓ · tests ✓   (or what failed)
  notes:   <risks / did-NOT-do / follow-ups — one line>

<repo-slug-2> ...

Jira: commented on {{PROJECT_KEY}}-#### with branch names + PR links   (or "skipped (--no-jira)")
Next: human reviews the PRs → approve → merge in promotion order (develop → staging → main).
```
If the ticket was too vague to implement: emit the one-line reason + the open questions, and confirm **no PR was opened**.

---

## Safety boundaries (the trust spine — PR is the only gate)
- **Never merge, never approve, never force-push, never modify pre-existing branches.** The bot only creates and pushes its own `{{PROJECT_KEY}}-####` branches and opens PRs; the human approves, decides promotion order, and merges.
- **Worktrees only** — never write in the main clones.
- **`main` (production) PRs are always `⚠️`**, and so is any high-risk change (DB migrations, dependency bumps, infra/CI changes, secret handling, large deletions) even on green.
- **Promotions stay faithful** — a staging/main PR must carry the *same* behaviour reviewed on develop. Port to fit a drifted branch, but never let the change quietly diverge.
- **No empty PRs** — skip an environment that already has the change.
- **Never fabricate verification.** Can't run it → "Unverified", say why.
- **Scope discipline** — implement the ticket, not adjacent ideas. Out-of-scope improvements go in "follow-ups", not the diff.
- **Stop rather than guess** — a vague ticket gets an open-questions note, not a speculative PR.

---

## Architecture and feature-to-repo map

**Read `ARCHITECTURE.md` at the root of this skills directory.** It names the
services, their stacks, the events and queues between them, and which feature
lives where. Fill it in once per organisation; every skill reads the same file.

Routing rule of thumb: pick the one to three most likely services before
reading any code, and when a symptom crosses a boundary, suspect the boundary
itself. Integration seams are where these bugs live.

## Coordinates
- Jira: project `{{PROJECT_KEY}}`, cloud `{{JIRA_HOST}}`. Use `mcp__atlassian-remote__*` (canonical; not the deprecated `claude_ai_Atlassian`).
- Bitbucket: workspace `{{WORKSPACE}}`, `mcp__bitbucket__*`. Remote = `git@bitbucket.org:{{WORKSPACE}}/<slug>.git`.
- AC field: `{{AC_FIELD_ID}}`. Issue types: Story 10018, Task 10019, Bug 10020.

## Prerequisite — Bitbucket write access
Opening PRs needs the **Bitbucket MCP credential to have `pullrequest:write`** (and repository read). The token lives in `BITBUCKET_PASSWORD` of the `bitbucket` server in `~/.claude.json`.
- **Symptom of a missing scope:** reads work (listing PRs, diffs, reviewers) but `createPullRequest` fails with **HTTP 403**. Everything up to the PR still succeeds, so the branch is pushed — only the PR step is blocked.
- **Fix:** give the credential PR-write scope — either a Bitbucket **App Password** with *Pull requests: Write*, or an Atlassian **API token** created with `write:pullrequest:bitbucket` — and update `BITBUCKET_PASSWORD`.
- **After updating the token, reload the MCP** so the running process picks it up: `/mcp` → reconnect **bitbucket** (no session restart needed), or restart Claude Code. A live session keeps the old token until reconnected, so a retry before reloading will 403 again — that is the stale process, not a bad scope.

## Fit with the other skills
`ticket-writer` (create a ticket) → **`dev-ticket` (implement → PR)** → `qa-code-review` / `qa-execute` (QA the PR). And `bug-hunter --ticket` → ticket → `dev-ticket`. This skill never reviews or QAs its own work beyond the mandatory self-review — that's the human's and the QA skills' job.
