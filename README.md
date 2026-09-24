# Claude Code skills for software delivery

Ten skills that take a piece of work from a symptom or a brief all the way to a
reviewed, tested pull request. They were built and used daily on a real product
team, then stripped of anything specific to that company so they can be pointed
at another one.

## The skills

| Skill | What it does |
|---|---|
| `bug-hunter` | Works backwards from a symptom to ranked root causes, with evidence. Routes to the likely service first, reads the code, checks the git history, and grounds the blast radius in data. |
| `ticket-writer` | Turns a rough brief into a clean Story, Task or Bug with business-outcome acceptance criteria. Drafts, confirms, then creates. |
| `dev-ticket` | Implements a ticket in an isolated worktree, runs the real checks, self-reviews adversarially, and opens a PR. The PR is the only human gate. |
| `code-explorer` | Multi-agent review of a ticket and its PRs, straight to the terminal. Four specialists, then two challengers. |
| `qa-orchestration` | Sequences the three QA skills below across a list of tickets, unattended. |
| `qa-code-review` | The same multi-agent review, written up as a deploy verdict for a non-engineer. |
| `qa-test-cases` | Generates test cases from acceptance criteria, diffs and review findings, then classifies which are safe to automate. |
| `qa-execute` | Runs the automatable cases in a browser and records pass or fail honestly. |
| `dev-performance-data` | Pulls sprint history and changelogs for delivery analytics. |

## Setup

1. Copy the skill directories into `.claude/skills/` in your project.
2. Fill in `CONFIG.md`. It lists every `{{PLACEHOLDER}}` and what it means.
3. Fill in `ARCHITECTURE.md`. The routing skills read it to pick a target
   service before they read any code, so the feature-to-service table is the
   part that earns its keep.

Nothing else needs editing.

## What they assume

- An issue tracker, source hosting with pull requests, and MCP servers for both.
- Optionally a wiki for QA output, and optionally a read-only data warehouse.
  Both are genuinely optional. Every skill that touches the warehouse is written
  to skip silently when it is missing.

## Ideas worth keeping if you rewrite them

A few things in here were learned the hard way and are the reason the skills
behave well rather than just verbosely.

**Challenge your own findings.** The review skills run four specialists and then
two challengers whose entire job is to attack the first four: one hunting false
positives and overstated severity, one filtering for what would actually cause
an incident. Most of the quality comes from the second round, not the first.

**An automation safety boundary.** `qa-test-cases` marks a case automatable only
when it is both frontend and non-destructive. Everything else is explicitly
manual. `qa-execute` then refuses to perform a step that would persist a change
even if a case told it to, and reports the misclassification. Two independent
checks, because the cost of an agent mutating shared state is not symmetric with
the cost of a skipped test.

**Never fake green.** `dev-ticket` reports the real test result, including
"could not verify", and puts what it did not do in the PR body. A confident
wrong status is worse than an honest gap.

**Absence is not evidence.** Anything reading from a warehouse mirror is warned
that the mirror lags, so a missing row usually means unsynced rather than
nonexistent. This one specific trap wasted enough time to earn a paragraph in
four separate skills.

**Write the verdict for whoever has to act on it.** The QA pages use plain words
and plain status labels, with code confined to collapsed blocks. A review nobody
reads has no value.

## Provenance

Point-in-time extraction from a private working set, genericised for reuse. Not
a maintained fork of anything.
