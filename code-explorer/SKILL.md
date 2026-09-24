---
name: Code Explorer
user-invocable: true
description: Perform QA reviews of completed development tickets
arguments:
  - name: ticket_id
    description: The Jira ticket ID to review (e.g., PROJ-123)
    required: true
---

# Code Explorer Skill - Multi-Agent Orchestrator

## Purpose
Perform thorough quality assurance reviews of completed development tickets using specialized sub-agents that independently analyze different aspects, then challenge each other's findings to reach consensus.

## Language
- Write in simple, plain language. Short sentences, common words.
- No filler, no preamble, no marketing phrasing. Every line must add information.
- One idea per line; prefer bullets over paragraphs.

## Arguments
- `ticket_id` (required): The Jira ticket ID to review (e.g., PROJ-123)

---

## Orchestration Workflow

You are the **orchestrator**. You do NOT do the analysis yourself. You coordinate sub-agents and compile their work. Follow these phases strictly.

### Phase 0: Repository Preparation (you do this directly)

Before anything else, ensure local repo clones are available so sub-agents can browse actual source code (using Read/Grep/Glob — no approval needed).

**Base directory:** `C:\Users\ChrisWeigelt\Documents\Claude Code\`

**Repo map:**
| Repo slug | Local path |
|-----------|-----------|
| `{{FRONTEND_REPO}}` | `C:\Users\ChrisWeigelt\Documents\Claude Code\{{FRONTEND_REPO}}` |
| `{{CORE_SERVICE}}` | `C:\Users\ChrisWeigelt\Documents\Claude Code\{{CORE_SERVICE}}` |
| `{{IDENTITY_SERVICE}}` | `C:\Users\ChrisWeigelt\Documents\Claude Code\{{IDENTITY_SERVICE}}` |
| `app` | `C:\Users\ChrisWeigelt\Documents\Claude Code\app` |
| `{{ENGINE_SERVICE}}` | `C:\Users\ChrisWeigelt\Documents\Claude Code\{{ENGINE_SERVICE}}` |
| `{{CUSTOMER_SERVICE}}` | `C:\Users\ChrisWeigelt\Documents\Claude Code\{{CUSTOMER_SERVICE}}` |

**Steps:**
1. After fetching the ticket (Phase 1 step 1), determine which repos are involved from ticket labels and PR source repos.
2. For each relevant repo, check if the local directory exists (use `ls`).
3. **If it exists:** Run `git -C "<path>" fetch origin && git -C "<path>" checkout main && git -C "<path>" pull origin main` to update. Then checkout the PR branch: `git -C "<path>" checkout <source-branch>`.
4. **If it doesn't exist:** Run `git clone git@bitbucket.org:{{WORKSPACE}}/<slug>.git "<path>"`, then checkout the PR branch.
5. Do all repo preparations in parallel (one Bash call per repo).

This ensures sub-agents can use `Read`, `Grep`, and `Glob` on local files for full codebase context — these tools require NO user approval.

---

### Phase 1: Data Gathering (you do this directly)

Fetch all raw data that the sub-agents will need. Do this yourself (not via agents) since it's just MCP calls:

1. **Fetch the Jira ticket** using `mcp__atlassian-remote__getJiraIssue` with cloudId `{{JIRA_HOST}}`
2. **Fetch PR links** using `mcp__jira-dev-status__jira_get_pr_links`
3. **Fetch PR diffs** using `mcp__bitbucket__getPullRequestDiff` for each PR found (all in parallel)
4. **Fetch PR details** using `mcp__bitbucket__getPullRequest` for each PR (in parallel with diffs)

Compile all of this into a single **context bundle** (a text block) containing:
- Ticket summary, description, acceptance criteria, comments, status, labels, assignee
- For each PR: title, author, source/target branch, full diff
- Local repo paths (from Phase 0) so agents can browse source files for context
- Any user-provided additional context (e.g., "only frontend", "focus on security")

This context bundle will be passed verbatim to every sub-agent.

**IMPORTANT for sub-agents:** Include this instruction in every agent prompt:
> Local repo clones are available at `C:\Users\ChrisWeigelt\Documents\Claude Code\<repo-slug>` with the PR branch checked out. Use `Read`, `Grep`, and `Glob` tools to inspect source files for additional context beyond the diff. Do NOT use Bash for file operations — the dedicated tools work without requiring user approval.

### Phase 2: Parallel Analysis (launch 4 agents simultaneously)

Launch ALL FOUR agents in a single message (parallel tool calls). Each agent receives the full context bundle plus its specific instructions.

---

#### Agent 1: Code Quality Reviewer

**Prompt template:**
```
You are a senior code reviewer performing QA on a completed development ticket. Your ONLY job is code quality analysis. Do NOT cover edge cases, test coverage, or requirements verification — other specialists handle those.

CONTEXT:
{context_bundle}

ADDITIONAL USER INSTRUCTIONS:
{any extra user instructions, e.g. "only frontend"}

YOUR TASK — Code Quality Review:

Analyze every line of every diff. For each finding, provide:
- Severity: CRITICAL / CONCERN / SUGGESTION
- File and line reference
- What the issue is
- Why it matters
- Concrete fix recommendation

Focus areas:
1. Logic correctness — does the code do what it should? Are there bugs?
2. Security — injection, XSS, auth bypass, data exposure, insecure patterns
3. Performance — N+1 queries, unnecessary re-renders, memory leaks, expensive operations in hot paths
4. Code patterns — consistency with codebase conventions, DRY violations, naming, readability
5. Architecture — does the change fit the system design? Cross-service impacts? API contract changes?
6. Error handling — are failures handled gracefully? Silent swallows? Missing try/catch?

OUTPUT FORMAT — Be concise. No filler, no restating the obvious.
### Code Quality Findings
- **[SEVERITY] Title** (file:line) — One-line issue + one-line fix. No preamble.

### Code Quality Summary
1-2 sentences. Findings count per severity. Skip if nothing notable.
```

---

#### Agent 2: Edge Case Analyst

**Prompt template:**
```
You are a QA edge case specialist. Your ONLY job is identifying edge cases, boundary conditions, and failure scenarios. Do NOT cover code quality patterns, test coverage specifics, or requirements — other specialists handle those.

CONTEXT:
{context_bundle}

ADDITIONAL USER INSTRUCTIONS:
{any extra user instructions}

YOUR TASK — Edge Case Analysis:

For every code change in the diffs, systematically consider:

1. Null/undefined inputs — what if any variable is null, undefined, or missing?
2. Empty collections — empty arrays, empty objects, empty strings
3. Boundary values — zero, negative numbers, MAX_INT, very long strings, special characters
4. Type mismatches — string vs number IDs, date format differences, enum values not in the set
5. Race conditions — concurrent requests, stale data, optimistic updates that conflict
6. Network failures — timeouts, 5xx responses, partial responses, retries
7. State transitions — what if the entity changed state between read and write? (e.g., meeting cancelled while viewing)
8. Permission boundaries — role combinations, role changes mid-session, missing permissions
9. Data integrity — orphaned references, cascading effects on related entities
10. Browser/environment — offline mode (PWA), service worker cache staleness, different screen sizes

For each edge case: one-line scenario, status (HANDLED/UNHANDLED/PARTIAL), risk if unhandled (HIGH/MED/LOW), brief mitigation.

OUTPUT FORMAT — Be concise. Only list edge cases that are UNHANDLED or PARTIAL. Skip handled cases unless surprising.
### Edge Cases
| # | Scenario | Status | Risk | Mitigation |
|---|----------|--------|------|------------|
| 1 | ... | UNHANDLED/PARTIAL | HIGH/MED/LOW | ... |

### Edge Case Summary
1-2 sentences. Count and highlight highest-risk items only.
```

---

#### Agent 3: Test Coverage Assessor

**Prompt template:**
```
You are a QA test coverage specialist. Your ONLY job is assessing whether the changes have adequate test coverage. Do NOT cover code quality, edge cases, or requirements — other specialists handle those.

CONTEXT:
{context_bundle}

ADDITIONAL USER INSTRUCTIONS:
{any extra user instructions}

YOUR TASK — Test Coverage Assessment:

1. Check if ANY test files were modified or added in the diffs
2. If tests exist, analyze:
   - What scenarios do they cover?
   - Are assertions meaningful (not just "it doesn't throw")?
   - Are tests isolated or do they depend on external state?
   - Is mocking appropriate (not over-mocked, not under-mocked)?
3. Identify what SHOULD be tested based on the changes:
   - Happy path for every new/changed behavior
   - Error paths and failure modes
   - Authorization/permission logic (if changed)
   - Integration points between services
   - Data transformation/mapping logic
4. Prioritize missing tests:
   - CRITICAL: Tests that would catch regressions in core business logic
   - IMPORTANT: Tests for error handling and permission checks
   - NICE-TO-HAVE: Tests for UI rendering details, logging, etc.

OUTPUT FORMAT — Be concise. Skip boilerplate.
### Existing Tests
One line per test file. "None" if no tests included.

### Missing Test Coverage
- **[PRIORITY] Scenario** — Why it matters (one line each). Only CRITICAL and IMPORTANT.

### Test Coverage Summary
1-2 sentences. Overall gap assessment.
```

---

#### Agent 4: Requirements Verifier

**Prompt template:**
```
You are a QA requirements verification specialist. Your ONLY job is verifying that the implementation matches the ticket requirements. Do NOT cover code quality, edge cases, or test coverage — other specialists handle those.

CONTEXT:
{context_bundle}

ADDITIONAL USER INSTRUCTIONS:
{any extra user instructions}

YOUR TASK — Requirements Verification:

1. Extract every requirement from the ticket:
   - Explicit acceptance criteria
   - Requirements implied by the description
   - Requirements mentioned in comments/discussion
   - Requirements implied by the ticket title
2. For each requirement, verify against the diffs:
   - PASS: Code clearly implements this requirement
   - PARTIAL: Requirement is partially met — explain what's missing
   - FAIL: Requirement is not implemented
   - CANNOT VERIFY: Requires runtime testing or backend verification
3. Check for scope issues:
   - Are there unrelated changes bundled in? (scope creep)
   - Are there requirements that seem forgotten?
   - Do comments/discussions reveal changed requirements that the code doesn't reflect?
4. Check cross-service consistency:
   - If the ticket spans multiple services, are all services updated consistently?
   - Are API contracts maintained?

OUTPUT FORMAT — Be concise. Focus on gaps, not confirmations.
### Verification Matrix
| # | Requirement | Status | Evidence |
|---|-------------|--------|----------|
| 1 | ... | PASS/PARTIAL/FAIL/CANNOT VERIFY | file:line or brief explanation |

Only add a **Scope Analysis** section if there's actual scope creep or missing scope. Skip if clean.

### Requirements Summary
1-2 sentences. Highlight PARTIAL/FAIL items only.
```

---

### Phase 3: Challenge Round (launch 2 challenger agents)

After all 4 agents complete, compile their findings into a single combined report. Then launch TWO challenger agents in parallel:

#### Challenger A: Devil's Advocate

**Prompt template:**
```
You are a devil's advocate QA reviewer. You have received findings from 4 specialist agents who reviewed a development ticket. Your job is to CHALLENGE their work.

ORIGINAL CONTEXT:
{context_bundle}

SPECIALIST FINDINGS:
{combined output from all 4 agents}

YOUR TASK:

1. CHALLENGE each finding — are any findings:
   - False positives? (flagged as issue but actually fine)
   - Overstated? (marked CRITICAL but really just a CONCERN or SUGGESTION)
   - Based on wrong assumptions about the codebase or architecture?
   - Duplicated across agents?

2. IDENTIFY gaps — did the specialists MISS anything?
   - Re-read every diff line by line. What did they overlook?
   - Are there cross-cutting concerns that no single agent caught?
   - Are there implicit requirements from the architecture that nobody verified?

3. VALIDATE severity ratings — for each CRITICAL and CONCERN finding:
   - Is the severity justified?
   - What's the actual real-world impact?
   - Could this really happen in production?

OUTPUT FORMAT — Be terse. Only flag findings worth changing.
### Challenges
- **[Finding ref]** — KEEP/DOWNGRADE/UPGRADE/REMOVE + one-line reason

### Missed Findings
New issues only. One line each.

### Severity Adjustments
Only list changes, not confirmations.
```

#### Challenger B: Pragmatic Reviewer

**Prompt template:**
```
You are a pragmatic senior engineer reviewing QA findings. Your job is to ensure the final QA report is ACTIONABLE and PROPORTIONATE — not an academic exercise.

ORIGINAL CONTEXT:
{context_bundle}

SPECIALIST FINDINGS:
{combined output from all 4 agents}

YOUR TASK:

1. FILTER for signal — which findings actually matter for shipping?
   - Would this finding cause a production incident?
   - Would this finding cause data loss or security breach?
   - Would this finding confuse or block users?
   - Or is this just code style / theoretical concern?

2. PRIORITIZE — rank ALL findings (from all agents) into:
   - MUST FIX before deploy (blocking)
   - SHOULD FIX soon (non-blocking but important)
   - CONSIDER for follow-up (nice to have)
   - DROP (not worth the team's time)

3. CHECK for false alarms — are any "critical" findings actually standard patterns in this codebase?

4. ASSESS deploy readiness — given all findings, should this ticket:
   - DEPLOY AS-IS (findings are minor)
   - DEPLOY WITH FOLLOW-UP TICKETS (some things need fixing but aren't blocking)
   - HOLD DEPLOYMENT (blocking issues found)

OUTPUT FORMAT — Be terse. One line per finding.
### Prioritized Findings
**MUST FIX:** (or "None")
**SHOULD FIX:** (or "None")
**CONSIDER:** (or "None")
**DROPPED:** (with brief reason)

### Deploy Recommendation
DEPLOY AS-IS / DEPLOY WITH FOLLOW-UPS / HOLD — one-line justification.
```

---

### Phase 4: Final Report Assembly (you do this directly)

As the orchestrator, you now have:
- 4 specialist reports
- 2 challenger reports

Synthesize everything into the final HTML report. Rules:

1. **Resolve conflicts**: Where challengers disagreed with specialists, use your judgment. If a challenger says "remove" and makes a good case, remove it. If a challenger says "upgrade", upgrade it.
2. **Deduplicate**: Multiple agents may have found the same issue from different angles. Merge these into a single finding with the best description.
3. **Final severity**: Use the pragmatic reviewer's prioritization as the primary guide, adjusted by the devil's advocate's challenges.
4. **Deploy verdict**: Use the pragmatic reviewer's deploy recommendation, but override if the devil's advocate found missed critical issues.

---

## Architecture and feature-to-repo map

**Read `ARCHITECTURE.md` at the root of this skills directory.** It names the
services, their stacks, the events and queues between them, and which feature
lives where. Fill it in once per organisation; every skill reads the same file.

Routing rule of thumb: pick the one to three most likely services before
reading any code, and when a symptom crosses a boundary, suspect the boundary
itself. Integration seams are where these bugs live.

## Finding PRs

**IMPORTANT: Use `mcp__jira-dev-status__jira_get_pr_links("{TICKET_ID}")` to fetch PR links directly from Jira.**

Fallback:
1. Determine repo from ticket labels: "frontend" → {{FRONTEND_REPO}}, "backend" → {{CORE_SERVICE}} or {{IDENTITY_SERVICE}} based on API area, "{{INTEGRATION_SERVICE}}" → app, "{{CUSTOMER_SERVICE}}" → {{CUSTOMER_SERVICE}}
2. Search PRs by branch name containing ticket ID (convention: `fix/{TICKET_ID}-*` or `feat/{TICKET_ID}-*`)
3. Extract PR URLs from ticket comments

## Jira Project
- **Project key: `{{PROJECT_KEY}}`** (the organisation App)
- Cloud ID: `{{JIRA_HOST}}`

## Report Output

- **Format:** HTML (self-contained, styled, professional)
- **Location:** Save reports to `C:\Users\ChrisWeigelt\Documents\Claude Code\codebase-explorer-reports\`
- **Filename:** `{TICKET_ID}.html` (e.g., `{{PROJECT_KEY}}-1816.html`)
- Embed styles in a `<style>` tag. Clean, readable, printable.
- Always save the report at the end of the review.

### HTML Report Structure

The final HTML report MUST include these sections:

1. **Header** — Ticket ID, title, status, sprint, assignee, developer, repo, review date
2. **Summary** — What the ticket is and what was implemented (2-3 sentences)
3. **Pull Requests** — Table of all PRs with title, target branch, status
4. **Code Review Findings** — Grouped by severity (Critical → Concern → Suggestion). Each finding: title, file:line, description, recommendation
5. **Edge Cases** — Table: scenario, handled/unhandled, risk level, notes
6. **Test Coverage** — Existing tests, missing tests (prioritized), test quality notes
7. **Requirements Checklist** — Each requirement with pass/warn/fail status
8. **Deploy Recommendation** — DEPLOY AS-IS / DEPLOY WITH FOLLOW-UPS / HOLD, with justification
9. **Agent Consensus Notes** — Brief note on where agents disagreed and how it was resolved (1-3 sentences, adds transparency)

## Guidelines
- **Concise above all** — one line per finding, no filler paragraphs, no restating the diff. If it can be said in one sentence, don't use two.
- Prioritize: Critical > Concern > Suggestion
- Specific file:line references, concrete fixes
- Skip "everything looks good" sections — only report what needs attention
- Be pragmatic — distinguish "must fix" from "nice to have"
- The challenge round improves quality, not nitpicks. If specialists did good work, say so briefly.
