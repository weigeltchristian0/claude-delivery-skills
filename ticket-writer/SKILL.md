---
name: Ticket Writer
user-invocable: true
description: Write concise, well-structured {{PROJECT_KEY}} Jira tickets (Story, Task, or Bug) from a rough brief. Use whenever the user wants to create, draft, or write a Jira ticket/issue/story/task/bug for the organisation's app — e.g. "write a ticket for X", "create a story about Y", "file a bug for Z", "draft a task to add column W". Produces a status-tagged user story (Story) or structured Problem/Solution body (Task/Bug), with business-oriented acceptance criteria placed in the dedicated Acceptance Criteria field. Drafts first, confirms, then creates in Jira. Runs code analysis via the Code Explorer skill only when the brief needs facts from the codebase.
arguments:
  - name: brief
    description: Rough description of what the ticket is about. Free text. If omitted, ask the user for it.
    required: false
  - name: type
    description: "Force the issue type: --type=Story | Task | Bug. If omitted, infer from the brief and confirm."
    required: false
  - name: draft-only
    description: "If set (--draft-only), produce the ticket text only and do NOT create it in Jira."
    required: false
---

# Ticket Writer

## Purpose
Turn a rough brief into a clean {{PROJECT_KEY}} ticket. Be terse. No filler, no preamble, no marketing language. Every line earns its place.

## Voice rules (apply to all output)
- Plain, direct sentences. Cut "in order to", "basically", "please note", "as we know".
- One idea per line. Prefer bullets over paragraphs.
- Concrete nouns (table, field, endpoint, role) over vague ones ("the system", "things").
- Don't restate the title in the body. Don't pad acceptance criteria to look thorough.

## Technical details (when included)
Include technical detail only where the developer needs it to start. Keep it as short, labelled bullets — never dense prose paragraphs. If there's nothing concrete to name, leave technical details out rather than filling space.

**Good — short labelled bullets:**
```
## Technical Details
- **API Gateway Base Path:** /customer
- **Response Format:** JSON:API
- **Related:** {{PROJECT_KEY}}-1608 — unified customer IDs
```

**Too much — rewrite as bullets:**
```
Develop REST API endpoints for customer data management with proper HTTP
semantics, request validation, and response formatting. The API must support
pagination, filtering, and sorting while maintaining consistent error handling
and comprehensive OpenAPI documentation.
```

## Jira coordinates
- Cloud: `{{JIRA_HOST}}` · Project key: `{{PROJECT_KEY}}`
- **Use the `mcp__atlassian-remote__*` MCP server** for all Jira calls (`createJiraIssue`, `editJiraIssue`, etc.) — it's the canonical one; do **not** use the deprecated `claude_ai_Atlassian` connector (its SSE transport is being retired).
- Issue types: `Story` (id 10018), `Task` (id 10019), `Bug` (id 10020)
- **Acceptance Criteria field: `{{AC_FIELD_ID}}`** (text area) — the AC ALWAYS goes here, never inside the Description.

## Workflow

### 1. Get the brief
If `brief` is empty, ask the user what the ticket is about. Don't proceed on a guess.

### 2. Pick the issue type
Use `--type` if given. Otherwise infer and state your choice in one line:
- **Bug** — something is broken / behaves wrong / produces bad data.
- **Story** — new user-facing capability framed as a user goal.
- **Task** — technical work with no direct user-goal framing (schema change, migration, refactor, infra).

### 3. Analyse code only if needed
Most tickets are written from the brief alone. Invoke the **Code Explorer** skill *only* when the ticket depends on facts you can't get from the user — exact table/column/field names, current behaviour of a function, which service owns a flow. Don't explore for tickets that are purely product/behavioural. Keep findings to what the ticket needs.

### 4. Draft (always show before creating)
Write the three parts: **Summary**, **Description**, **Acceptance Criteria**. Follow the per-type templates below. Show the full draft to the user and ask for sign-off. Do not create the issue until they approve (unless `--draft-only`, in which case stop here and output the text).

### 5. Create in Jira (after approval)
What accepts what:
- **Description** — Task/Bug bodies are plain markdown (fine via `contentFormat: "markdown"`). A **Story** description has status lozenges, so it **must be ADF** (see the Story template).
- **Acceptance Criteria** (`{{AC_FIELD_ID}}`) — rejects markdown, **always ADF** (a `bulletList`).

Steps:
1. `createJiraIssue` — `projectKey: {{PROJECT_KEY}}`, `issueTypeName`, `summary`, and `description`. Use `contentFormat: "markdown"` for Task/Bug; for a Story pass the ADF description and `contentFormat: "adf"`.
2. `editJiraIssue` on the new key — `contentFormat: "adf"`, `fields: { "{{AC_FIELD_ID}}": <ADF doc> }`, AC as an ADF `bulletList`:
   ```json
   {"type":"doc","version":1,"content":[{"type":"bulletList","content":[
     {"type":"listItem","content":[{"type":"paragraph","content":[{"type":"text","text":"<criterion>"}]}]}
   ]}]}
   ```

When **filling an existing empty ticket** instead of creating one, skip step 1 and `editJiraIssue` the description and AC on that key. Description and AC are separate `editJiraIssue` calls when they need different `contentFormat`.

Then report the key + URL (`https://{{JIRA_HOST}}/browse/<KEY>`) and nothing else.

---

## Summary line
One line, ≤ ~80 chars, no trailing period. Lead with the concrete object.
- Story: capability — `Let team leads approve overtime requests`
- Task: action + object — `Add cancelled_by column to meetings table for booking app`
- Bug: scope: symptom — `Bookings by a newly promoted user attribute to the wrong owner id`

---

## Description templates

### Story — user story with status tags + context
Three inline status lozenges carry As / I want / Because, each followed by the sentence, then an optional context paragraph:

```
[User]  As a <role>,
[need]  I want <capability>,
[goal]  Because <business reason>.

<1–3 sentence context: why now, constraint, or background. Optional — omit if the story is self-evident.>
```

The role is a real the organisation actor ({{SPECIALIST_ROLE}}, internal sales, customer service agent, admin, customer). The "Because" is the business reason, not a restatement of the capability.

**Important — lozenges must be ADF, not markdown.** When *reading* a Jira issue you'll see lozenges as `<custom data-type="status" ...>User</custom>` — that is a display artifact. Writing that string produces literal text, not a lozenge. To create real lozenges, write the **whole description as ADF** with `status` inline nodes (colors: `User`=blue, `need`=yellow, `goal`=green):
```json
{"type":"doc","version":1,"content":[
  {"type":"paragraph","content":[
    {"type":"status","attrs":{"localId":"id-0","text":"User","color":"blue","style":""}},
    {"type":"text","text":" As a <role>,"}]},
  {"type":"paragraph","content":[
    {"type":"status","attrs":{"localId":"id-1","text":"need","color":"yellow","style":""}},
    {"type":"text","text":" I want <capability>,"}]},
  {"type":"paragraph","content":[
    {"type":"status","attrs":{"localId":"id-2","text":"goal","color":"green","style":""}},
    {"type":"text","text":" Because <business reason>."}]},
  {"type":"paragraph","content":[{"type":"text","text":"<context>"}]}
]}
```

### Task — statement + Problem/Solution
```
<One-sentence statement of the work.>

**Problem:** <why this is needed — the current gap or pain.>

**Solution:** <what to do, concretely. Name tables/columns/endpoints.>
```
Drop a section only if it's genuinely empty. Keep technical detail to what the developer needs to start.

### Bug — Problem / Steps / Impact (+ guard if recurring)
```
## Problem
<What's wrong, with the concrete evidence: IDs, codes, values, the verified case.>

## Fix
<Numbered, concrete remediation steps.>

## Permanent guard   ← include only if this is a recurring pattern
<The systemic fix and prior recurrences.>

## Impact
<Who/what is affected and any deadline that makes it urgent.>
```
Always ground a bug in specifics (row IDs, field values, the one case you verified). Vague bugs get bounced.

---

## Acceptance Criteria (field `{{AC_FIELD_ID}}`)
Business-oriented checklist. Each line is an observable outcome a non-engineer could confirm — what's true when the work is done, not how it's built. Lightly technical only where a name removes ambiguity (a column, a role, a screen). Use a plain bullet list (`-`), not checkboxes. 3–7 items is usually right; don't pad.

**Good (Story):**
```
- An {{SPECIALIST_ROLE}} can book a meeting in a slot outside their working hours.
- The booking is recorded as voluntary additional work, not company overtime.
- Service and Nachanpassung meeting types are bookable this way.
- Working-hours rules for normal (non-voluntary) booking are unchanged.
```

**Good (Task):**
```
- A `cancelled_by` column on the meetings table records who cancelled.
- A companion column records the cancellation timestamp.
- The value is set at cancellation time and not overwritten by later updates.
```

**Good (Bug):**
```
- A promoted user's bookings attribute to their own owner id, not the default.
- Her 45 affected bookings since 2026-05-11 are re-synced and re-attributed.
- Future bookings under code `ANSP` resolve to her active employee row.
```

Avoid: implementation steps, restating the title, "works correctly", or criteria a tester couldn't check.

For a **Task or Bug** where outcomes are already fully covered by the Description's Solution/Fix, AC can be brief or omitted — say so rather than padding. Stories should always have AC.

---

## Output discipline
- Before create: show Summary / Description / Acceptance Criteria, then one question — "Create this in {{PROJECT_KEY}}?"
- After create: one line — `Created {{PROJECT_KEY}}-#### — <url>`.
- No status narration, no "I'll now…". Just the draft, the confirm, the result.
